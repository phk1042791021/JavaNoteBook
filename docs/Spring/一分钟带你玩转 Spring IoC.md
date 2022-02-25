## 是何

上一篇文章有同学问我在官网该看哪些内容，怎么找的，那今天的截图里都会有链接。

### 初识 IoC

根据上一篇文章我们说的，Spring 全家桶中最重要的几个项目都是基于 Spring Framework 的，所以我们就以 Spring Framework 为例来看**文档**[2]。

首先它的右侧有 Github 的链接，另外点到「LEARN」这里，就会看到各个版本的文档。

![图片](/Users/yunai/编程/JavaNoteBook/docs/Spring/images/640-20220217152152070)

那我们点「Reference Doc」，就能够看到它的一些模块的介绍：

（等下... 模块？什么是模块？这个问题下文回答。）

![图片](/Users/yunai/编程/JavaNoteBook/docs/Spring/images/640-20220217152215171)

第一章 Overview，讲述它的历史、设计原理等等；

第二章 Core，包含了 IoC 容器，AOP 等等，那自然是讲 Spring 的核心了，要点进去好好看了。

![图片](/Users/yunai/编程/JavaNoteBook/docs/Spring/images/640-20220217152219048)

点进去之后发现了宝贵的学习资料，一切的 what, why, how 都可以在这里找到答案。

![图片](/Users/yunai/编程/JavaNoteBook/docs/Spring/images/640-20220217152222768)

这里很好的解释了大名鼎鼎的 IoC - Inversion of Control, 控制反转。

我粗略的总结一下：控制反转就是把创建和管理 bean 的过程转移给了第三方。而这个第三方，就是 Spring IoC Container，对于 IoC 来说，最重要的就是**容器**。

容器负责创建、配置和管理 bean，也就是它管理着 bean 的生命，控制着 bean 的依赖注入。

通俗点讲，因为项目中每次创建对象是很麻烦的，所以我们使用 Spring IoC 容器来管理这些对象，需要的时候你就直接用，不用管它是怎么来的、什么时候要销毁，只管用就好了。

举个例子，就好像父母没时间管孩子，就把小朋友交给托管所，就安心的去上班而不用管孩子了。托儿所，就是第三方容器，负责管理小朋友的吃喝玩乐；父母，相当于程序员，只管接送孩子，不用管他们吃喝。

等下，`bean` 又是什么？

Bean 其实就是包装了的 Object，无论是控制反转还是依赖注入，它们的主语都是 object，而 bean 就是由第三方包装好了的 object。（想一下别人送礼物给你的时候都是要包装一下的，自己造的就免了。

Bean 是 Spring 的主角，有种说法叫 Spring 就是面向 bean 的编程（Bean Oriented Programming, BOP）。

### IoC 容器

既然说容器是 IoC 最重要的部分，那么 Spring 如何设计容器的呢？还是回到官网，第二段有介绍哦：

![图片](/Users/yunai/编程/JavaNoteBook/docs/Spring/images/640-20220217152227450)

答：使用 `ApplicationContext`，它是 `BeanFactory` 的子类，更好的补充并实现了 `BeanFactory` 的。

`BeanFactory` 简单粗暴，可以理解为 HashMap：

- Key - bean name
- Value - bean object

但它一般只有 get, put 两个功能，所以称之为“低级容器”。

而 `ApplicationContext` 多了很多功能，因为它继承了多个接口，可称之为“高级容器”。在下文的搭建项目中，我们会使用它。

![图片](/Users/yunai/编程/JavaNoteBook/docs/Spring/images/640)

`ApplicationContext` 的里面有两个具体的实现子类，用来读取配置配件的：

- `ClassPathXmlApplicationContext` - 从 class path 中加载配置文件，更常用一些；
- `FileSystemXmlApplicationContext` - 从本地文件中加载配置文件，不是很常用，如果再到 Linux 环境中，还要改路径，不是很方便。

当我们点开 `ClassPathXmlApplicationContext` 时，发现它并不是直接继承 `ApplicationContext` 的，它有很多层的依赖关系，每层的子类都是对父类的补充实现。

而再往上找，发现最上层的 class 回到了 `BeanFactory`，所以它非常重要。

要注意，Spring 中还有个 `FactoryBean`，两者并没有特别的关系，只是名字比较接近，所以不要弄混了顺序。

为了好理解 IoC，我们先来回顾一下不用 IoC 时写代码的过程。

### 深入理解 IoC

这里用经典 `class Rectangle` 来举例：

- 两个变量：长和宽
- 自动生成 `set()` 方法和 `toString()` 方法

注意 ⚠️：一定要生成 `set()` 方法，因为 Spring IoC 就是通过这个 `set()` 方法注入的；`toString()` 方法是为了我们方便打印查看。

```java
public class Rectangle {
    private int width;
    private int length;

    public Rectangle() {
        System.out.println("Hello World!");
    }


    public void setWidth(int widTth) {
        this.width = widTth;
    }

    public void setLength(int length) {
        this.length = length;
    }

    @Override
    public String toString() {
        return "Rectangle{" +
                "width=" + width +
                ", length=" + length +
                '}';
    }
}
```

然后在 `test` 文件中手动用 `set()` 方法给变量赋值。

嗯，其实这个就是「解藕」的过程！

```java
public class MyTest {
  @Test
  public void myTest() {
    Rectangle rect = new Rectangle();
    rect.setLength(2);
    rect.setWidth(3);
    System.out.println(rect);
  }
}
```

其实这就是 IoC 给属性赋值的实现方法，我们把「创建对象的过程」转移给了 `set()` 方法，而不是靠自己去 `new`，就不是自己创建的了。

这里我所说的“自己创建”，指的是直接在对象内部来 `new`，是程序主动创建对象的正向的过程；这里使用 `set()` 方法，是别人（test）给我的；而 IoC 是用它的容器来创建、管理这些对象的，其实也是用的这个 `set()` 方法，不信，你把这个这个方法去掉或者改个名字试试？

#### 几个关键问题：

**何为控制，控制的是什么？**

答：是 bean 的创建、管理的权利，控制 bean 的整个生命周期。

**何为反转，反转了什么？**

答：把这个权利交给了 Spring 容器，而不是自己去控制，就是反转。由之前的自己主动创建对象，变成现在被动接收别人给我们的对象的过程，这就是反转。

举个生活中的例子，主动投资和被动投资。

自己炒股、选股票的人就是主动投资，主动权掌握在自己的手中；而买基金的人就是被动投资，把主动权交给了基金经理，除非你把这个基金卖了，否则具体选哪些投资产品都是基金经理决定的。

### 依赖注入

回到文档中，第二句话它说：`IoC is also known as DI`.

我们来谈谈 `dependency injection` - 依赖注入。

**何为依赖，依赖什么？**

程序运行需要依赖外部的资源，提供程序内对象的所需要的数据、资源。

**何为注入，注入什么？**

配置文件把资源从外部注入到内部，容器加载了外部的文件、对象、数据，然后把这些资源注入给程序内的对象，维护了程序内外对象之间的依赖关系。

所以说，控制反转是通过依赖注入实现的。但是你品，你细品，它们是有差别的，像是`「从不同角度描述的同一件事」`：

- IoC 是设计思想，DI 是具体的实现方式；
- IoC 是理论，DI 是实践；

从而实现对象之间的解藕。

**当然，IoC 也可以通过其他的方式来实现，而 DI 只是 Spring 的选择。**

IoC 和 DI 也并非 Spring 框架提出来的，Spring 只是应用了这个设计思想和理念到自己的框架里去。

## 为何

那么为什么要用 IoC 这种思想呢？换句话说，IoC 能给我们带来什么好处？

答：解藕。

它把对象之间的依赖关系转成用配置文件来管理，由 Spring IoC Container 来管理。

在项目中，底层的实现都是由很多个对象组成的，对象之间彼此合作实现项目的业务逻辑。但是，很多很多对象紧密结合在一起，一旦有一方出问题了，必然会对其他对象有所影响，所以才有了解藕的这种设计思想。

![图片](/Users/yunai/编程/JavaNoteBook/docs/Spring/images/640-20220217152320204)

![图片](/Users/yunai/编程/JavaNoteBook/docs/Spring/images/640-20220217152320198)

如上图所示，本来 ABCD 是互相关联在一起的，当加入第三方容器的管理之后，每个对象都和第三方法的 IoC 容器关联，彼此之间不再直接联系在一起了，没有了耦合关系，全部对象都交由容器来控制，降低了这些对象的亲密度，就叫“解藕”。

## 如何

最后到了实践部分，我们来真的搭建一个 Spring 项目，使用下 IoC 感受一下。

现在大都使用 `maven` 来构建项目，方便我们管理 jar 包；但我这里先讲一下手动导入 jar 包的过程，中间会遇到很多问题，都是很好的学习机会。

在开始之前，我们先来看下图 - 大名鼎鼎的 Spring 模块图。

### Spring Framework 八大模块

![图片](/Users/yunai/编程/JavaNoteBook/docs/Spring/images/640-20220217152320230)

模块化的思想是 Spring 中非常重要的思想。

Spring 框架是一个分层架构，每个模块既可以单独使用，又可与其他模块联合使用。

每个「绿框」，对应一个模块，总共8个模块；「黑色包」，表示要实现这个模块的 jar 包。

`Core Container`，我们刚才已经在文档里看到过了，就是 IoC 容器，是核心，可以看到它依赖于这4个 jar 包：

- `Beans`
- `Core`
- `Context`
- `SpEL`, spring express language

那这里我们就知道了，如果想要用 IoC 这个功能，需要把这 4个 jar 包导进去。其中，Core 模块是 Spring 的核心，Spring 的所有功能都依赖于这个 jar 包，Core 主要是实现 IoC 功能，那么说白了 Spring 的所有功能都是借助于 IoC 实现的。

其他的模块和本文关系不大，不在这里展开了。

那当我们想搭建 Spring 项目时，当然可以把所有 jar 包都导进去，但是你的电脑能受得了吗。。 但是包越大，项目越大，问题就越多，所以尽量按需选择，不用囤货。。

Btw, 这张图在网上有很多，但是在我却没有在最新版的 reference doc 上找到。。不过，既然那些老的教程里有，说明老版本的 doc 里有，那去**老版本的介绍**[3] 里找找看😂

在本文第一张图 `Spring Framework` - `Documentation` 中我们选 `4.3.26` 的 `Reference Doc.`，然后搜索“`Framework Modules`”，就有啦～ 具体链接可以看文末参考资料。

还有一个方法，待会我们讲到 jar 包中的内容时再说。

## 小结

我们最后再来体会一下用 Spring 创建对象的过程：

通过 `ApplicationContext` 这个 IoC 容器的入口，用它的两个具体的实现子类，从 class path 或者 file path 中读取数据，用 `getBean()` 获取具体的 bean instance。

那使用 Spring 到底省略了我们什么工作？

答：`new 的过程`。把 new 的过程交给第三方来创建、管理，这就是「解藕」。

![图片](/Users/yunai/编程/JavaNoteBook/docs/Spring/images/640-20220217152454151)

Spring 也是用的 `set()` 方法，它只不过提供了一套更加完善的实现机制而已。

而说到底，底层的原理并没有很复杂，只是为了提高扩展性、兼容性，Spring 提供了丰富的支持，所以才觉得源码比较难。

因为框架是要给各种各样的用户来使用的，它们考虑的更多的是扩展性。如果让我们来实现，或许三五行就能搞定，但是我们实现的不完善、不完整、不严谨，总之不高大上，所以它写三五十行，把框架设计的尽可能的完善，提供了丰富的支持，满足不同用户的需求，才能占领更大的市场啊。