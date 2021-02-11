## 查找表

* 指某个**集合需要进行查找**
  * 如果集合元素，那么 Set
  * 如果集合元素还有其对应的值，那么用 map



### Set

* 对于某个集合要经常用来查找或者去重，那么使用 Set



#### [349. 两个数组的交集](https://leetcode-cn.com/problems/intersection-of-two-arrays/)

难度简单321收藏分享切换为英文接收动态反馈

给定两个数组，编写一个函数来计算它们的交集。

 

**示例 1：**

```
输入：nums1 = [1,2,2,1], nums2 = [2,2]
输出：[2]
```

**示例 2：**

```
输入：nums1 = [4,9,5], nums2 = [9,4,9,8,4]
输出：[9,4]
```

 

**题解**

```java
class Solution {
    public int[] intersection(int[] nums1, int[] nums2) {
        Set<Integer> set = new HashSet<>();
        List<Integer> resList = new ArrayList<>();
        
        for(int i : nums1){
            set.add(i);
        }

        for(int i : nums2){
            if(set.contains(i)){
                resList.add(i);
                set.remove(i);
            }
        }

        int[] res = new int[resList.size()];
        for(int i = 0 ; i < resList.size(); i ++){
            res[i] = resList.get(i);
        }

        return res;
    }
}
```



#### [219. 存在重复元素 II](https://leetcode-cn.com/problems/contains-duplicate-ii/)

难度简单238收藏分享切换为英文接收动态反馈

给定一个整数数组和一个整数 *k*，判断数组中是否存在两个不同的索引 *i* 和 *j*，使得 **nums [i] = nums [j]**，并且 *i* 和 *j* 的差的 **绝对值** 至多为 *k*。

 

**示例 1:**

```
输入: nums = [1,2,3,1], k = 3
输出: true
```

**示例 2:**

```
输入: nums = [1,0,1,1], k = 1
输出: true
```

**示例 3:**

```
输入: nums = [1,2,3,1,2,3], k = 2
输出: false
```



**题解**

要使用之前的 n 个元素，那么可以维护一个有 n 个元素的窗口。如果这个窗口还要用来查找，那么 Set 就非常合适

```java
class Solution {
    public boolean containsNearbyDuplicate(int[] nums, int k) {
        Set<Integer> record = new HashSet<>();
         
        for(int i = 0 ; k != 0 && i < nums.length ; i ++){
            if(record.contains(nums[i])){
                return true;
            }


            if(record.size() == k){
                record.remove(nums[i - k]);
            }
            record.add(nums[i]);
        }

        return false;
    }
}
```

### 滑动窗口

#### [217. 存在重复元素](https://leetcode-cn.com/problems/contains-duplicate/)

难度简单360收藏分享切换为英文接收动态反馈

给定一个整数数组，判断是否存在重复元素。

如果存在一值在数组中出现至少两次，函数返回 `true` 。如果数组中每个元素都不相同，则返回 `false` 。

 

**示例 1:**

```
输入: [1,2,3,1]
输出: true
```

**示例 2:**

```
输入: [1,2,3,4]
输出: false
```



**题解**

```java
class Solution {
    public boolean containsDuplicate(int[] nums) {
        Set<Integer> record = new HashSet<>();
        
        for(int i : nums){
            if(record.contains(i)){
                return true;
            }
            record.add(i);
        }

        return false;
    }
}
```



#### [220. 存在重复元素 III](https://leetcode-cn.com/problems/contains-duplicate-iii/)

难度中等293收藏分享切换为英文接收动态反馈

在整数数组 `nums` 中，是否存在两个下标 ***i\*** 和 ***j\***，使得 **nums [i]** 和 **nums [j]** 的差的绝对值小于等于 ***t*** ，且满足 ***i\*** 和 ***j\*** 的差的绝对值也小于等于 ***ķ*** 。

如果存在则返回 `true`，不存在返回 `false`。

 

**示例 1:**

```
输入: nums = [1,2,3,1], k = 3, t = 0
输出: true
```

**示例 2:**

```
输入: nums = [1,0,1,1], k = 1, t = 2
输出: true
```

**示例 3:**

```
输入: nums = [1,5,9,1,5,9], k = 2, t = 3
输出: false
```



**双层for 超时**

```java
class Solution {
   public boolean containsNearbyAlmostDuplicate(int[] nums, int k, int t) {
        int res = 0;
        for(int i = 0 ; i < nums.length ;  i ++){
            for(int j = i - 1 ; j >= Math.max(i - k, 0) ; j --){
                long d = (long)nums[i] - (long)nums[j];
                if(Math.abs(d) <= t){
                    res ++;
                }
            }
        }

        return res != 0;
    }
}
```



**TreeSet 作窗口**

```java
class Solution {
   public boolean containsNearbyAlmostDuplicate(int[] nums, int k, int t) {
       TreeSet<Long> record = new TreeSet<>();
       int recordSize = 0;
       for(int i = 0 ; i < nums.length ; i ++){
           Long ceiling = record.ceiling((long)nums[i] - t);
           if(ceiling != null && ceiling <= (long)nums[i] + t){
               return true;
           }
           
           if(record.size() == k){
               record.remove((long)nums[i - k]);
           }
           if(k != 0){
               record.add((long)nums[i]);
           }
       }

       return false;
    }
}
```

