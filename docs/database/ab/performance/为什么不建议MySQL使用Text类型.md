## 前言

众所周知，MySQL广泛应用于互联网的OLTP（联机事务处理过程）业务系统中，在大厂开发规范中，经常会看到一条"不建议使用text大字段类型”。

下面就从text类型的存储结构，引发的问题解释下为什么不建议使用text类型，以及Text改造的建议方法。

## 背景

### 写log表导致DML慢

#### 问题描述

某歪有一个业务系统，使用RDS for MySQL 5.7的高可用版本，配置long_query_time=1s，添加慢查询告警，我第一反应就是某歪又乱点了。

我通过监控看CPU， QPS，TPS等指标不是很高，最近刚好双十一全站都在做营销活动，用户量稍微有所增加。某歪反馈有些原本不慢的接口变的很慢，影响了正常的业务，需要做一下troubleshooting。

#### 问题分析

我从慢查询告警，可以看到有一些insert和update语句比较慢，同时告警时段的监控，发现IOPS很高，达到了70MB/s左右，由于RDS的CloundDBA功能不可用，又没有audit log功能，troubleshooting比较困难，硬着头皮只能分析binlog了。

配置了max_binlog_size =512MB，在IOPS高的时段里，看下binlog的生成情况。

![图片](images/640-20220104134810325)

需要分析为什么binlog写这么快，最有可能原因就是insert into request_log表上有text类型，request_log表结构如下（demo）

```sql
CREATE TABLE request_log (`
 `id bigint(20) NOT NULL AUTO_INCREMENT,`
 `log text,`    
 `created_at datetime NOT NULL,`
 `status tinyint(4) NOT NULL,`
 `method varchar(10) DEFAULT NULL,`
 `url varchar(50) DEFAULT NULL,`
 `update_at datetime DEFAULT NULL,`
 `running_time tinyint(4) DEFAULT '0',`
 `user_id bigint(20) DEFAULT NULL,`
 `type varchar(50) DEFAULT NULL,`
 `PRIMARY KEY (id)`
`) ENGINE=InnoDB AUTO_INCREMENT=4229611 DEFAULT CHARSET=utf8`
```

分析binlog：

```bash
$ mysqlbinlog --no-defaults -v -v --base64-output=DECODE-ROWS mysql-bin.000539|egrep "insert into request_log"
```

满屏幕都是看不清的内容，翻了半天没翻完。

基本上已经确定是写入request_log的log字段引起的，导致binlog_cache频繁的flush，以及binlog过度切换，导致IOPS过高，影响了其他正常的DML操作。

#### 问题解决

跟开发同学沟通后，计划在下一个版本修复这个问题，不再将request信息写入表中，写入到本地日志文件，通过filebeat抽取到es进行查询，如果只是为了查看日志也可以接入grayLog等日志工具，没必要写入数据库。

文章最后我还会介绍几个MySQL 我踩过Text相关的坑，这介绍坑之前我先介绍下MySQLText类型。

## MySQL中的Text

### Text类型

text是一个能够存储大量的数据的大对象，有四种类型：TINYTEXT, TEXT, MEDIUMTEXT,LONGTEXT，不同类型存储的值范围不同，如下所示

| Data Type  | Storage Required             |
| :--------- | :--------------------------- |
| TINYTEXT   | L + 1 bytes, where L < 2**8  |
| TEXT       | L + 2 bytes, where L < 2**16 |
| MEDIUMTEXT | L + 3 bytes, where L < 2**24 |
| LONGTEXT   | L + 4 bytes, where L < 2**32 |

其中L表是text类型中存储的实际长度的字节数。可以计算出TEXT类型最大存储长度2**16-1 = 65535 Bytes。

### InnoDB数据页

Innodb数据页由以下7个部分组成：

| 内容                        | 占用大小 | 说明                                                         |
| :-------------------------- | :------- | :----------------------------------------------------------- |
| File Header                 | 38Bytes  | 数据文件头                                                   |
| Page Header                 | 56 Bytes | 数据页头                                                     |
| Infimun 和 Supermum Records |          | 伪记录                                                       |
| User Records                |          | 用户数据                                                     |
| Free Space                  |          | 空闲空间：内部是链表结构，记录被delete后，会加入到free_lru链表 |
| Page  Dictionary            |          | 页数据字典：存储记录的相对位置记录，也称为Slot，内部是一个稀疏目录 |
| File Trailer                | 8Bytes   | 文件尾部：为了检测页是否已经完整个的写入磁盘                 |

说明：File Trailer只有一个FiL_Page_end_lsn部分，占用8字节，前4字节代表该页的checksum值，最后4字节和File Header中的FIL_PAGE_LSN，一个页是否发生了Corrupt，是通过File Trailer部分进行检测，而该部分的检测会有一定的开销，用户可以通过参数innodb_checksums开启或关闭这个页完整性的检测。

从MySQL 5.6开始默认的表存储引擎是InnoDB，它是面向ROW存储的，每个page(default page size = 16KB)，存储的行记录也是有规定的，最多允许存储16K/2 - 200 = 7992行。

### InnoDB的行格式

Innodb支持四种行格式：

| 行格式     | Compact存储特性 | 增强的变长列存储 | 支持大前缀索引 | 支持压缩 | 支持表空间类型                  |
| :--------- | :-------------- | :--------------- | :------------- | :------- | :------------------------------ |
| REDUNDANT  | No              | No               | No             | No       | system, file-per-table, general |
| COMPACT    | Yes             | No               | No             | No       | system, file-per-table, general |
| DYNAMIC    | Yes             | Yes              | Yes            | No       | system, file-per-table, general |
| COMPRESSED | Yes             | Yes              | Yes            | Yes      | file-per-table, general         |

由于Dynamic是Compact变异而来，结构大同而已，现在默认都是Dynamic格式；COMPRESSED主要是对表和索引数据进行压缩，一般适用于使用率低的归档，备份类的需求，主要介绍下REDUNDANT和COMPACT行格式。

#### Redundant行格式

这种格式为了兼容旧版本MySQL。

**行记录格式：**

| Variable-length offset list | record_header        | col1_value | col2_value | …….  | text_value     |
| :-------------------------- | :------------------- | :--------- | :--------- | :--- | :------------- |
| 字段长度偏移列表            | 记录头信息，占48字节 | 列1数据    | 列2数据    | …….  | Text列指针数据 |

**具有以下特点：**

- 存储变长列的前768 Bytes在索引记录中，剩余的存储在overflow page中，对于固定长度且超过768 Bytes会被当做变长字段存储在off-page中。
- 索引页中的每条记录包含一个6 Bytes的头部，用于链接记录用于行锁。
- 聚簇索引的记录包含用户定义的所有列。另外还有一个6字节的事务ID（DB_TRX_ID）和一个7字节长度的回滚段指针(Roll pointer)列。
- 如果创建表没有显示指定主键，每个聚簇索引行还包括一个6字节的行ID（row ID）字段。
- 每个二级索引记录包含了所有定义的主键索引列。
- 一条记录包含一个指针来指向这条记录的每个列，如果一条记录的列的总长度小于128字节，这个指针占用1个字节，否则2个字节。这个指针数组称为记录目录（record directory）。指针指向的区域是这条记录的数据部分。
- 固定长度的字符字段比如CHAR(10)通过固定长度的格式存储，尾部填充空格。
- 固定长度字段长度大于或者等于768字节将被编码成变长的字段，存储在off-page中。
- 一个SQL的NULL值存储一个字节或者两个字节在记录目录（record dirictoty）。对于变长字段null值在数据区域占0个字节。对于固定长度的字段，依然存储固定长度在数据部分，为null值保留固定长度空间允许列从null值更新为非空值而不会引起索引的分裂。
- 对varchar类型，Redundant行记录格式同样不占用任何存储空间，而CHAR类型的NULL值需要占用空间。

其中变长类型是通过长度 + 数据的方式存储，不同类型长度是从1到4个字节（L+1 到 L + 4），对于TEXT类型的值需要L Bytes存储value，同时需要2个字节存储value的长度。同时Innodb最大行长度规定为65535 Bytes，对于Text类型，只保存9到12字节的指针，数据单独存在overflow page中。

#### Compact行格式

这种行格式比redundant格式减少了存储空间作为代价，但是会增加某些操作的CPU开销。如果系统workload是受缓存命中率和磁盘速度限制，compact行格式可能更快。如果你的工作负载受CPU速度限制，compact行格式可能更慢，Compact 行格式被所有file format所支持。

**行记录格式：**

| Variable-length field length list | NULL标志位 | record_header | col1_value | col2_value | …….  | text_value     |
| :-------------------------------- | :--------- | :------------ | :--------- | :--------- | :--- | :------------- |
| 变长字段长度列表                  |            | 记录头信息-   | 列1数据    | 列2数据    | …….  | Text列指针数据 |

Compact首部是一个非NULL变长字段长度的列表，并且是按列的顺序逆序放置的，若列的长度小于255字节，用1字节表示；若大于255个字节，用2字节表示。变长字段最大不可以超过2字节，这是因为MySQL数据库中varchar类型最大长度限制为65535，变长字段之后的第二个部分是NULL标志位，表示该行数据是否有NULL值。有则用1表示，该部分所占的字节应该为1字节。

所以在创建表的时候，尽量使用NOT NULL DEFAULT ''，如果表中列存储大量的NULL值，一方面占用空间，另一个方面影响索引列的稳定性。

**具有以下特点：**

- 索引的每条记录包含一个5个字节的头部，头部前面可以有一个可变长度的头部。这个头部用来将相关连的记录链接在一起，也用于行锁。
- 记录头部的变长部分包含了一个表示null 值的位向量(bit vector)。如果索引中可以为null的字段数量为N，这个位向量包含 N/8 向上取整的字节数。比例如果有9-16个字段可以为NULL值，这个位向量使用两个字节。为NULL的列不占用空间，只占用这个位向量中的位。头部的变长部分还包含了变长字段的长度。每个长度占用一个或者2个字节，这取决了字段的最大长度。如果所有列都可以为null 并且制定了固定长度，记录头部就没有变长部分。
- 对每个不为NULL的变长字段，记录头包含了一个字节或者两个字节的字段长度。只有当字段存储在外部的溢出区域或者字段最大长度超过255字节并且实际长度超过127个字节的时候会使用2个字节的记录头部。对应外部存储的字段，两个字节的长度指明内部存储部分的长度加上指向外部存储部分的20个字节的指针。内部部分是768字节，因此这个长度值为 768+20， 20个字节的指针存储了这个字段的真实长度。
- NULL不占该部分任何空间，即NULL除了占用NULL标志位，实际存储不占任何空间。
- 记录头部跟着非空字段的数据部分。
- 聚簇索引的记录包含了所以用户定于的字段。另外还有一个6字节的事务ID列和一个7字节的回滚段指针。
- 如果没有定于主键索引，则聚簇索引还包括一个6字节的Row ID列。
- 每个辅助索引记录包含为群集索引键定义的不在辅助索引中的所有主键列。如果任何一个主键列是可变长度的，那么每个辅助索引的记录头都有一个可变长度的部分来记录它们的长度，即使辅助索引是在固定长度的列上定义的。
- 固定长度的字符字段比如CHAR(10)通过固定长度的格式存储，尾部填充空格。
- 对于变长的字符集，比如uft8mb3和utf8mb4， InnoDB试图用N字节来存储 CHAR(N)。如果CHAR(N)列的值的长度超过N字节，列后面的空格减少到最小值。CHAR(N)列值的最大长度是最大字符编码数 x N。比如utf8mb4字符集的最长编码为4，则列的最长字节数是 4*N。

## Text类型引发的问题

### 插入text字段导致报错

#### 创建测试表

```sql
[root@barret] [test]>create table user(id bigint not null primary key auto_increment, 
  -> name varchar(20) not null default '' comment '姓名', 
  -> age tinyint not null default 0 comment 'age', 
  -> gender char(1) not null default 'M' comment '性别',
  -> info text not null comment '用户信息',
  -> create_time datetime NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  -> update_time datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '修改时间'
  -> );
Query OK, 0 rows affected (0.04 sec)
```

#### 插入测试数据

```sql
root@barret] [test]>insert into user(name,age,gender,info) values('moon', 34, 'M', repeat('a',1024*1024*3));
ERROR 1406 (22001): Data too long for column 'info' at row 1
[root@barret] [test]>insert into user(name,age,gender,info) values('sky', 35, 'M', repeat('b',1024*1024*5));
ERROR 1301 (HY000): Result of repeat() was larger than max_allowed_packet (4194304) - truncated
```

#### 错误分析

```sql
[root@barret] [test]>select @@max_allowed_packet;
+----------------------+
| @@max_allowed_packet |
+----------------------+
|       4194304 |
+----------------------+
1 row in set (0.00 sec)
```

max_allowed_packet控制communication buffer最大尺寸，当发送的数据包大小超过该值就会报错，我们都知道，MySQL包括Server层和存储引擎，它们之间遵循2PC协议，Server层主要处理用户的请求：连接请求—>SQL语法分析—>语义检查—>生成执行计划—>执行计划—>fetch data；存储引擎层主要存储数据，提供数据读写接口。

```bash
max_allowed_packet=4M，当第一条insert repeat('a',1024*1024*3)，数据包Server执行SQL发送数据包到InnoDB层的时候，检查数据包大小没有超过限制4M，在InnoDB写数据时，发现超过了Text的限制导致报错。第二条insert的数据包大小超过限制4M，Server检测不通过报错。
```

**引用AWS RDS参数组中该参数的描述**

max_allowed_packet:  This value by default is small, to catch large (possibly incorrect) packets. Must be increased if using large TEXT columns or long strings. As big as largest BLOB.

增加该参数的大小可以缓解报错，但是不能彻底的解决问题。

### RDS实例被锁定

#### 背景描述

公司每个月都会做一些营销活动，有个服务apush活动推送，单独部署在高可用版的RDS for MySQL 5.7，配置是4C8G 150G磁盘，数据库里也就4张表，晚上22：00下班走的时候，rds实例数据使用了50G空间，第二天早晨9：30在地铁上收到钉钉告警短信，提示push服务rds实例由于disk is full被locked with —read-only，开发也反馈，应用日志报了一堆MySQL error。

#### 问题分析

通过DMS登录到数据库，看一下那个表最大，发现有张表push_log占用了100G+，看了下表结构，里面有两个text字段。

```sql
request text default '' comment '请求信息',
response text default '' comment '响应信息'
mysql>show  table status like 'push_log'；
```

发现Avg_row_length基本都在150KB左右，Rows = 78w，表的大小约为780000*150KB/1024/1024 = 111.5G。

### 通过主键update也很慢

```sql
insert into user(name,age,gender,info) values('thooo', 35, 'M', repeat('c',65535);
insert into user(name,age,gender,info) values('thooo11', 35, 'M', repeat('d',65535);
insert into user(name,age,gender,info) select name,age,gender,info from user;
Query OK, 6144 rows affected (5.62 sec)
Records: 6144  Duplicates: 0  Warnings: 0                                        
[root@barret] [test]>select count(*) from user;
+----------+
| count(*) |
+----------+
|    24576 |
+----------+
1 row in set (0.05 sec)
```

做update操作并跟踪。

```sql
mysql> set profiling = 1;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> update user set info = repeat('f',65535) where id = 11;
Query OK, 1 row affected (0.28 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> show profiles;
+----------+------------+--------------------------------------------------------+
| Query_ID | Duration   | Query                                                  |
+----------+------------+--------------------------------------------------------+
|        1 | 0.27874125 | update user set info = repeat('f',65535) where id = 11 |
+----------+------------+--------------------------------------------------------+
1 row in set, 1 warning (0.00 sec)

mysql> show profile cpu,block io for query 1;  
+----------------------+----------+----------+------------+--------------+---------------+
| Status               | Duration | CPU_user | CPU_system | Block_ops_in | Block_ops_out |
+----------------------+----------+----------+------------+--------------+---------------+
| starting             | 0.000124 | 0.000088 |   0.000035 |            0 |             0 |
| checking permissions | 0.000021 | 0.000014 |   0.000006 |            0 |             0 |
| Opening tables       | 0.000038 | 0.000026 |   0.000011 |            0 |             0 |
| init                 | 0.000067 | 0.000049 |   0.000020 |            0 |             0 |
| System lock          | 0.000076 | 0.000054 |   0.000021 |            0 |             0 |
| updating             | 0.244906 | 0.000000 |   0.015382 |            0 |         16392 |
| end                  | 0.000036 | 0.000000 |   0.000034 |            0 |             0 |
| query end            | 0.033040 | 0.000000 |   0.000393 |            0 |           136 |
| closing tables       | 0.000046 | 0.000000 |   0.000043 |            0 |             0 |
| freeing items        | 0.000298 | 0.000000 |   0.000053 |            0 |             0 |
| cleaning up          | 0.000092 | 0.000000 |   0.000092 |            0 |             0 |
+----------------------+----------+----------+------------+--------------+---------------+
11 rows in set, 1 warning (0.00 sec)
```

可以看到主要耗时在updating这一步，IO输出次数16392次，在并发的表上通过id做update，也会变得很慢。

### group_concat也会导致查询报错

在业务开发当中，经常有类似这样的需求，需要根据每个省份可以定点医保单位名称，通常实现如下：

```sql
select group_concat(dru_name) from t_drugstore group by province;
```

其中内置group_concat返回一个聚合的string，最大长度由参数group_concat_max_len（Maximum allowed result length in bytes for the GROUP_CONCAT()）决定，默认是1024，一般都太短了，开发要求改长一点，例如1024000。

当group_concat返回的结果集的大小超过max_allowed_packet限制的时候，程序会报错，这一点要额外注意。

### MySQL内置的log表

MySQL中的日志表mysql.general_log和mysql.slow_log，如果开启审计audit功能，同时log_output=TABLE，就会有mysql.audit_log表，结构跟mysql.general_log大同小异。

分别看一下他们的表结构

```sql
CREATE TABLE `general_log` (
  `event_time` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  `user_host` mediumtext NOT NULL,
  `thread_id` bigint(21) unsigned NOT NULL,
  `server_id` int(10) unsigned NOT NULL,
  `command_type` varchar(64) NOT NULL,
  `argument` mediumblob NOT NULL
) ENGINE=CSV DEFAULT CHARSET=utf8 COMMENT='General log'
CREATE TABLE `slow_log` (
  `start_time` timestamp(6) NOT NULL DEFAULT CURRENT_TIMESTAMP(6) ON UPDATE CURRENT_TIMESTAMP(6),
  `user_host` mediumtext NOT NULL,
  `query_time` time(6) NOT NULL,
  `lock_time` time(6) NOT NULL,
  `rows_sent` int(11) NOT NULL,
  `rows_examined` int(11) NOT NULL,
  `db` varchar(512) NOT NULL,
  `last_insert_id` int(11) NOT NULL,
  `insert_id` int(11) NOT NULL,
  `server_id` int(10) unsigned NOT NULL,
  `sql_text` mediumblob NOT NULL,
  `thread_id` bigint(21) unsigned NOT NULL
) ENGINE=CSV DEFAULT CHARSET=utf8 COMMENT='Slow log'
```

mysql.general_log记录的是经过MySQL Server处理的所有的SQL，包括后端和用户的，insert比较频繁，同时`argument` mediumblob NOT NULL，对MySQL Server性能有影响的，一般我们在dev环境为了跟踪排查问题，可以开启general_log，Production环境禁止开启general_log，可以开启audit_log，它是在general_log的基础上做了一些filter，比如我只需要业务账号发起的所有的SQL，这个很有用的，很多时候需要分析某一段时间内哪个SQL的QPS，TPS比较高。

mysql.slow_log记录的是执行超过long_query_time的所有SQL，如果遵循MySQL开发规范，slow query不会太多，但是开启了log_queries_not_using_indexes=ON就会有好多full table scan的SQL被记录，这时slow_log表会很大，对于RDS来说，一般只保留一天的数据，在频繁insert into slow_log的时候，做truncate table slow_log去清理slow_log会导致MDL，影响MySQL稳定性。

建议将log_output=FILE，开启slow_log， audit_log，这样就会将slow_log，audit_log写入文件，通过Go API处理这些文件将数据写入分布式列式数据库clickhouse中做统计分析。

## Text改造建议

### 使用es存储

在MySQL中，一般log表会存储text类型保存request或response类的数据，用于接口调用失败时去手动排查问题，使用频繁的很低。可以考虑写入本地log file，通过filebeat抽取到es中，按天索引，根据数据保留策略进行清理。

### 使用对象存储

有些业务场景表用到TEXT，BLOB类型，存储的一些图片信息，比如商品的图片，更新频率比较低，可以考虑使用对象存储，例如阿里云的OSS，AWS的S3都可以，能够方便且高效的实现这类需求。

## 总结

由于MySQL是单进程多线程模型，一个SQL语句无法利用多个cpu core去执行，这也就决定了MySQL比较适合OLTP（特点：大量用户访问、逻辑读，索引扫描，返回少量数据，SQL简单）业务系统，同时要针对MySQL去制定一些建模规范和开发规范，尽量避免使用Text类型，它不但消耗大量的网络和IO带宽，同时在该表上的DML操作都会变得很慢。

另外建议将复杂的统计分析类的SQL，建议迁移到实时数仓OLAP中，例如目前使用比较多的clickhouse，里云的ADB，AWS的Redshift都可以，做到OLTP和OLAP类业务SQL分离，保证业务系统的稳定性。