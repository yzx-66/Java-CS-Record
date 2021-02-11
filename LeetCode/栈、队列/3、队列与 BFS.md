### 队列

* 只要是树的层序遍历，遍历方法永远都是用 queue，不同题型变得只是添加元素方式（即 addFirst 还是 addLast）



#### [102. 二叉树的层序遍历](https://leetcode-cn.com/problems/binary-tree-level-order-traversal/)

难度中等765收藏分享切换为英文接收动态反馈

给你一个二叉树，请你返回其按 **层序遍历** 得到的节点值。 （即逐层地，从左到右访问所有节点）。

 

**示例：**
二叉树：`[3,9,20,null,null,15,7]`,

```
    3
   / \
  9  20
    /  \
   15   7
```

返回其层序遍历结果：

```
[
  [3],
  [9,20],
  [15,7]
]
```



**题解**

* 解法 1： 正常的深度优先遍历，但是状态维护 level，level 相同的加到同层 level 的 list 里面
* 解法 2：广度优先，用队列

```java
class Solution {
    public List<List<Integer>> levelOrder(TreeNode root) {
        List<List<Integer>> res = new LinkedList<>();
        if(root == null){
            return res;
        }

        TreeNode dummy = new TreeNode();
        LinkedList<TreeNode> queue = new LinkedList<>();
        queue.addLast(root);
        queue.addLast(dummy);

        while(queue.size() != 0){
            List<Integer> thisLevel = new LinkedList<>();
            TreeNode node;
            while((node = queue.pop()) != dummy){
                thisLevel.add(node.val);
                if(node.left != null){
                    queue.addLast(node.left);
                }
                if(node.right != null){
                    queue.addLast(node.right);
                }
            }
            if(queue.size() != 0){
                queue.addLast(dummy);
            }    
            res.add(thisLevel);
        }

        return res;
    }
}
```

#### [107. 二叉树的层序遍历 II](https://leetcode-cn.com/problems/binary-tree-level-order-traversal-ii/)

难度简单400收藏分享切换为英文接收动态反馈

给定一个二叉树，返回其节点值自底向上的层序遍历。 （即按从叶子节点所在层到根节点所在的层，逐层从左向右遍历）

例如：
给定二叉树 `[3,9,20,null,null,15,7]`,

```
    3
   / \
  9  20
    /  \
   15   7
```

返回其自底向上的层序遍历为：

```
[
  [15,7],
  [9,20],
  [3]
]
```



**题解**

```java
class Solution {
    public List<List<Integer>> levelOrderBottom(TreeNode root) {
        LinkedList<List<Integer>> res = new LinkedList<>();
        if(root == null){
            return res;
        }

        TreeNode dummy = new TreeNode();
        LinkedList<TreeNode> queue = new LinkedList<>();
        queue.addLast(root);
        queue.addLast(dummy);

        while(queue.size() != 0){
            TreeNode node;
            List<Integer> level = new LinkedList<>();
            while((node = queue.pop()) != dummy){
                level.add(node.val);
                if(node.left != null){
                    queue.addLast(node.left);
                }
                if(node.right != null){
                    queue.addLast(node.right);
                }
            }

            if(queue.size() != 0){
                queue.addLast(dummy);
            }
            // 这里，每次放到第一个
            res.addFirst(level);
        }

        return res;
    }
}
```



#### [103. 二叉树的锯齿形层序遍历](https://leetcode-cn.com/problems/binary-tree-zigzag-level-order-traversal/)

难度中等381收藏分享切换为英文接收动态反馈

给定一个二叉树，返回其节点值的锯齿形层序遍历。（即先从左往右，再从右往左进行下一层遍历，以此类推，层与层之间交替进行）。

例如：
给定二叉树 `[3,9,20,null,null,15,7]`,

```
    3
   / \
  9  20
    /  \
   15   7
```

返回锯齿形层序遍历如下：

```
[
  [3],
  [20,9],
  [15,7]
]
```



**题解**

````java
class Solution {
    public List<List<Integer>> zigzagLevelOrder(TreeNode root) {
        List<List<Integer>> res = new LinkedList<>();
        if(root == null){
            return res;
        }

        LinkedList<TreeNode> queue = new LinkedList<>();
        TreeNode dummy = new TreeNode();
        queue.addLast(root);
        queue.addLast(dummy);

        int level = 1;

        while(queue.size() != 0){
            TreeNode node;
            LinkedList<Integer> levelData = new LinkedList<>();
            while((node = queue.pop()) != dummy){
                if(level % 2 == 1){
                    levelData.addLast(node.val);
                }else {
                    levelData.addFirst(node.val);
                }

                if(node.left != null){
                    queue.addLast(node.left);
                }
                if(node.right != null){
                    queue.addLast(node.right);
                }
            }

            if(queue.size() != 0){
                queue.addLast(dummy);
            }
            level ++;
            res.add(levelData);
        }

        return res;
    }
}
````



#### [199. 二叉树的右视图](https://leetcode-cn.com/problems/binary-tree-right-side-view/)

难度中等399收藏分享切换为英文接收动态反馈

给定一棵二叉树，想象自己站在它的右侧，按照从顶部到底部的顺序，返回从右侧所能看到的节点值。

**示例:**

```
输入: [1,2,3,null,5,null,4]
输出: [1, 3, 4]
解释:

   1            <---
 /   \
2     3         <---
 \     \
  5     4       <---
```



**题解**

```java
class Solution {
    public List<Integer> rightSideView(TreeNode root) {
        List<Integer> res = new LinkedList<>();
        if(root == null){
            return res;
        }

        LinkedList<TreeNode> queue = new LinkedList<>();
        TreeNode dummy = new TreeNode();
        queue.add(root);
        queue.add(dummy);

        while(queue.size() != 0){
            TreeNode node;
            while((node = queue.pop()) != dummy){
                if(node.left != null){
                    queue.addLast(node.left);
                }
                if(node.right != null){
                    queue.addLast(node.right);
                }

                if(queue.getFirst() == dummy){
                    res.add(node.val);
                }
            }

            if(queue.size() != 0){
                queue.add(dummy);
            }
        }

        return res;
    }

   
}
```





#### [127. 单词接龙](https://leetcode-cn.com/problems/word-ladder/)

难度困难689收藏分享切换为英文接收动态反馈

字典 `wordList` 中从单词 `beginWord` 和 `endWord` 的 **转换序列** 是一个按下述规格形成的序列：

- 序列中第一个单词是 `beginWord` 。
- 序列中最后一个单词是 `endWord` 。
- 每次转换只能改变一个字母。
- 转换过程中的中间单词必须是字典 `wordList` 中的单词。

给你两个单词 `beginWord` 和 `endWord` 和一个字典 `wordList` ，找到从 `beginWord` 到 `endWord` 的 **最短转换序列** 中的 **单词数目** 。如果不存在这样的转换序列，返回 0。

**示例 1：**

```
输入：beginWord = "hit", endWord = "cog", wordList = ["hot","dot","dog","lot","log","cog"]
输出：5
解释：一个最短转换序列是 "hit" -> "hot" -> "dot" -> "dog" -> "cog", 返回它的长度 5。
```

**示例 2：**

```
输入：beginWord = "hit", endWord = "cog", wordList = ["hot","dot","dog","lot","log"]
输出：0
解释：endWord "cog" 不在字典中，所以无法进行转换。
```

 

**提示：**

- `1 <= beginWord.length <= 10`
- `endWord.length == beginWord.length`
- `1 <= wordList.length <= 5000`
- `wordList[i].length == beginWord.length`
- `beginWord`、`endWord` 和 `wordList[i]` 由小写英文字母组成
- `beginWord != endWord`
- `wordList` 中的所有字符串 **互不相同**



**题解**

对于状态转移中存在环的，那么求最短路径一定是 BFS + visit 标志数组 + level 计数

* 标志数组用来决定是否把某元素继续加入队列
* 对于 DFS 来说，用标志数组来递归，碰到有环（无向图、有向图存在闭合多边形）的情况，如果通过标志数组直接返回的话， 将无法获得该节点的返回值（因为该节点还正在向下递归，但是又绕回来了，没有机会递归到底再回溯）



**注意：对于图的 BFS 来说，仅适用于无权图（即可以理解为每个节点的值为 1），如果是有权图（即每个节点里的值不同），那么就不能用 BFS，必须使用 Floyd 算法（动态规划算法，适用于有向图与无向图）**

```java
public class Solution {

    public int ladderLength(String beginWord, String endWord, List<String> wordList) {
       LinkedList<String> queue = new LinkedList<>();
       queue.addLast(beginWord);

       int level = 1;
       HashSet<String> isVisit = new HashSet<>();

       while(queue.size() != 0){
           int levelSize = queue.size();
           for(int i = 0 ; i < levelSize ; i ++){
               String word = queue.pop();
               if(word.equals(endWord)){
                   return level;
               }

               for(int j = 0 ; j < wordList.size() ; j ++){
                   String w = wordList.get(j);
                   if(isVisit.contains(w)){
                       continue;
                   }

                   int dif = 0;
                   for(int k = 0 ;  k < word.length() ; k ++){
                       if(word.charAt(k) != w.charAt(k)){
                           dif ++;
                       }
                   }

                   if(dif == 1){
                       isVisit.add(w);
                       queue.addLast(w);
                   }
               }
           }

           level ++;
       }

       return 0;
    }

    
}
```



##### 有权图最短路径：Floyd 算法

* 每个状态有的决定因素有两个，起始节点和终止节点（即 dp 数组是二维的）

* 状态转移方程

  `fun(i , j) = max([i , j]  , max([i , k] + [k , j]))`

```c
typedef struct          {        
    char vertex[VertexNum];                                //顶点表         
    int edges[VertexNum][VertexNum];                       //邻接矩阵（对于两个边不可达的是无穷）        
    int n,e;                                               //图中当前的顶点数和边数         
}MGraph; 

void Floyd(MGraph g){
    int A[MAXV][MAXV]; // dp 数组
    int path[MAXV][MAXV]; // 路径（跟 dp 数组对应的，每个节点的下一步）
    int i,j,k,n=g.n;
    
    // 初始化 dp 数组（对于 dp 问题大多数都要初始化，或者多开辟一行）。
    for(i=0;i<n;i++){
        for(j=0;j<n;j++){ 　　
            A[i][j]=g.edges[i][j];
            path[i][j]=-1;
        }
    }
    
    // 常规动态规划的模板
    // 至于这个 k 为什么放最外层，根据状态方程 dij(k) = min(dij(k-1) , dik(k-1) + dkj(k-1)), k-1 代表不包含k时的值，
    // 可以理解为 k 是每次增加一个（即中间节点每次加一个），把 i 和 j 看成一个整体（代表 i 到 j）的长度。
    // 对于常规的，在状态转移方程里有 3 个变量，但是状态的决定属性只有两个，那么把状态多的那个属性放在最外层，把里面的二维看作一个整体，每次外层增加一个然后选择最优。
    // 跟背包问题其实很像，对于 k 都是只能选一次，因为一旦选了 k 作中转，后面如果再选 k 作中转，那么会形成环，是肯定不可达 i 或者 j 的，所以问题就变成了每个节点只可以选一次，那么只有选不选这两种情况，所以就还是背包问题。
    for(k=0;k<n;k++){ 
        for(i=0;i<n;i++){
            for(j=0;j<n;j++){
                if(A[i][j]>(A[i][k]+A[k][j])){                   　　
                    A[i][j]=A[i][k]+A[k][j];
                    path[i][j]=k;
                } 
            }
        } 
    } 
    
}
```



#### 小结

对于状态转移无环：DFS（记忆化搜索）

对于状态转移有环

* 无权：BFS（队列 + isVisit 标志）

* 有权：动态规划（把 i j 看成一个整体，i 和 j 代表两个顶点，可以是任意两个对象）

  * 其实对于常规的三层 for 的动态规划，基本都满足可以把其中两维看作一个整体
