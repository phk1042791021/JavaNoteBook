```java
class Solution {

    public boolean isSubStructure(TreeNode A, TreeNode B) {
        if(A == null || B==null){
            return false;
        }
        //寻找根节点
        Deque<TreeNode> queue = new LinkedList<>();
        queue.offer(A);
        TreeNode nodeA = null;
        TreeNode nodeB = null;
        while(!queue.isEmpty()){
            nodeA = queue.pop();
            if(nodeA.val == B.val){
                if(compare(nodeA,B)){
                    return true;
                }
                
                
            }
            if(nodeA.left != null){
                queue.offer(nodeA.left);
            }
            if(nodeA.right != null){
                queue.offer(nodeA.right);
            }

        }

    
        return false;
    }
    private boolean compare(TreeNode A, TreeNode B){
        if(A == null){
            return false;
        }

        //比较结构
        Deque<TreeNode> queue = new LinkedList<>();

        TreeNode nodeA = null;
        TreeNode nodeB = null;
        queue.offer(B);
        queue.offer(A);
        while(!queue.isEmpty()){
            nodeB = queue.pop();
            nodeA = queue.pop();

            if(nodeA.val != nodeB.val){
                return false;
            }
            if(nodeB.left != null){
                if(nodeA.left!=null && nodeB.left.val == nodeA.left.val){
                    queue.offer(nodeB.left);
                    queue.offer(nodeA.left);
                }else{
                    return false;
                }
                
            }
            if(nodeB.right != null){
                if(nodeA.right!=null && nodeB.right.val == nodeA.right.val){
                    queue.offer(nodeB.right);
                    queue.offer(nodeA.right);
                }else{
                    return false;
                }
            }

        }
        return true;
    }
}
```

