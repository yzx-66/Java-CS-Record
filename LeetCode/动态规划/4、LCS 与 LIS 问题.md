### LIS 问题

#### [300. 最长递增子序列](https://leetcode-cn.com/problems/longest-increasing-subsequence/)

给你一个整数数组 nums ，找到其中最长严格递增子序列的长度。

子序列是由数组派生而来的序列，删除（或不删除）数组中的元素而不改变其余元素的顺序。例如，[3,6,2,7] 是数组 [0,3,1,6,2,2,7] 的子序列。

示例 1：

```
输入：nums = [10,9,2,5,3,7,101,18]
输出：4
解释：最长递增子序列是 [2,3,7,101]，因此长度为 4 。
```


示例 2：

```
输入：nums = [0,1,0,3,2,3]
输出：4
```


示例 3：

```
输入：nums = [7,7,7,7,7,7,7]
输出：1
```

​                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         

**动态规划**

```java
class Solution {
     public int lengthOfLIS(int[] nums) {
        int[]  dp = new int[nums.length];
        

        for(int i = nums.length - 1 ;  i >= 0 ; i --){
            int max = 0;
            for(int j = i + 1; j < nums.length ; j ++){
                if(nums[i] < nums[j]){
                    max = Math.max(max , dp[j]);
                }
            }
            dp[i] = max + 1;
        }

        int res = 0;
        for(int i = 0 ; i < dp.length ; i ++){
            res = Math.max(res , dp[i]);
        }
        return res;
    }
}
```



**记忆化搜索**

```java
class Solution {
    int memo[];
    public int lengthOfLIS(int[] nums) {
        memo = new int[nums.length + 1];
        for(int i = 0 ; i <= nums.length ; i ++){
            memo[i] = -1;
        }

        int max = 0;
        for(int i = 0 ; i < nums.length ; i ++){
            max = Math.max(dfs(nums , i) , max);
        }
        return max;
    }

    int dfs(int nums[] , int idx){
        if(memo[idx] != -1){
            return memo[idx];
        }

        int max = 0;
        for(int i = idx + 1 ; i < nums.length ; i ++){
            if(nums[i] > nums[idx]){
                max = Math.max(max , dfs(nums , i));
            }
        }

        max = idx == nums.length ? 0 : max + 1;
        memo[idx] = max;
        return max;
    }
}
```



#### [376. 摆动序列](https://leetcode-cn.com/problems/wiggle-subsequence/)

如果连续数字之间的差严格地在正数和负数之间交替，则数字序列称为摆动序列。第一个差（如果存在的话）可能是正数或负数。少于两个元素的序列也是摆动序列。

例如， [1,7,4,9,2,5] 是一个摆动序列，因为差值 (6,-3,5,-7,3) 是正负交替出现的。相反, [1,4,7,2,5] 和 [1,7,4,5,5] 不是摆动序列，第一个序列是因为它的前两个差值都是正数，第二个序列是因为它的最后一个差值为零。

给定一个整数序列，返回作为摆动序列的最长子序列的长度。 通过从原始序列中删除一些（也可以不删除）元素来获得子序列，剩下的元素保持其原始顺序。

示例 1:

```
输入: [1,7,4,9,2,5]
输出: 6 
解释: 整个序列均为摆动序列。
```


示例 2:

```
输入: [1,17,5,10,13,15,10,5,16,8]
输出: 7
解释: 这个序列包含几个长度为 7 摆动序列，其中一个可为[1,17,10,13,10,16,8]。
```


示例 3:

```
输入: [1,2,3,4,5,6,7,8,9]
输出: 2
```



**记忆化搜索**

```
class Solution {
    int[][] memo ;
    public int wiggleMaxLength(int[] nums) {
        if(nums.length == 0){
            return 0;
        }
        memo = new int[2][nums.length];
        return func(nums , 0 , 1);
    }

    int func(int[] nums , int idx ,int isFatherBigger){
        if(memo[isFatherBigger][idx] != 0){
            return memo[isFatherBigger][idx];
        }

        int max = 0;
        if(idx == 0){
            for(int i = 1 ; i < nums.length ; i ++){
                if(nums[0] > nums[i] ){
                    max = Math.max(max , func(nums , i , 1) + 1);
                }
                if(nums[0] < nums[i] ){
                    max = Math.max(max , func(nums , i , 0) + 1);
                }
            }
        }else {
            for(int i = idx + 1  ; i < nums.length ; i ++){
                if(isFatherBigger == 1 && nums[idx] < nums[i]){
                    max = Math.max(max , func(nums , i , 0) + 1);
                }
                if(isFatherBigger == 0 && nums[idx] > nums[i]){
                    max = Math.max(max , func(nums , i , 1) + 1);
                }
            }
        }
        
        max = max ==  0 ? 1 : max;
        memo[isFatherBigger][idx] = max;
        return max;
    }
}
```



**动态规划**

```java
class Solution {
    public int wiggleMaxLength(int[] nums) {
       if(nums.length == 0){
           return 0;
       }
       int[][] dp = new int[nums.length][2];

        for(int i =  nums.length - 2 ; i >= 0 ; i --){
            for(int j = i + 1 ; j < nums.length  ; j ++){
                if(nums[i] > nums[j]){                
                    dp[i][1] = Math.max(dp[i][1] , dp[j][0] + 1);
                }
                if(nums[i] < nums[j]){
                    dp[i][0] = Math.max(dp[i][0] , dp[j][1] + 1);
                }
            }
        }

        return dp[0][0]  > dp[0][1] ? dp[0][0] + 1 : dp[0][1] + 1;
    }
}
```



### LCS 问题

* 对于最值问题，还是要先考虑动态规划，然后看如何进行子问题拆分 



![在这里插入图片描述](https://img-blog.csdnimg.cn/20210210231921944.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80MzkzNDYwNw==,size_16,color_FFFFFF,t_70)


```java
public class TestLCS {

    int testLCS(String s1 ,String s2){
        int[][] dp = new int[s1.length() + 1][s2.length() + 1];

        for(int i = s1.length() - 1; i >= 0 ; i --){
            for(int j = s2.length() - 1; j >= 0 ; j --){
                if(s1.charAt(i) == s2.charAt(j)){
                    dp[i][j] = dp[i + 1][j + 1] + 1;
                }else {
                    dp[i][j] = Math.max(dp[i + 1][j] , dp[i][j + 1]);
                }
            }
        }

        return dp[0][0];
    }
}
```

