### Map

#### [350. 两个数组的交集 II](https://leetcode-cn.com/problems/intersection-of-two-arrays-ii/)

难度简单443收藏分享切换为英文接收动态反馈

给定两个数组，编写一个函数来计算它们的交集。

 

**示例 1：**

```
输入：nums1 = [1,2,2,1], nums2 = [2,2]
输出：[2,2]
```

**示例 2:**

```
输入：nums1 = [4,9,5], nums2 = [9,4,9,8,4]
输出：[4,9]
```

 

**题解**

跟数据库原理中交运算的计算方法很类似

* 对于无序：hash 表 

* 对于有序：多路归并

```java
class Solution {
    public int[] intersect(int[] nums1, int[] nums2) {
        Map<Integer , Integer> map = new HashMap<>();
        List<Integer> resList = new ArrayList<>();

        for(int i : nums1){
            map.put(i , map.getOrDefault(i , 0) + 1);
        }
        for(int i : nums2){
            if(map.get(i) != null && map.get(i) != 0){
                map.put(i , map.get(i) - 1);
                resList.add(i);
            }
        }

        int[] res = new int[resList.size()];
        for(int i = 0 ;i < resList.size() ; i ++){
            res[i] = resList.get(i);
        }

        return res;
    }
}
```



#### [242. 有效的字母异位词](https://leetcode-cn.com/problems/valid-anagram/)

难度简单336收藏分享切换为英文接收动态反馈

给定两个字符串 *s* 和 *t* ，编写一个函数来判断 *t* 是否是 *s* 的字母异位词。

**示例 1:**

```
输入: s = "anagram", t = "nagaram"
输出: true
```

**示例 2:**

```
输入: s = "rat", t = "car"
输出: false
```



**题解**

```java
class Solution {
    public boolean isAnagram(String s, String t) {
        if(s.length() != t.length()){
            return false;
        }
        Map<Character , Integer> record = new HashMap<>();

        for(int i = 0 ; i < s.length() ; i ++){
            char c = s.charAt(i);
            record.put(c , record.getOrDefault(c , 0) + 1);
        }

        for(int i = 0 ;i < t.length() ; i ++){
            char c = t.charAt(i);
            if(record.get(c) != null && record.get(c) != 0){
                record.put(c , record.get(c) - 1);
            }else {
                return false;
            }
        }

        return true;
    }
}
```



#### [202. 快乐数](https://leetcode-cn.com/problems/happy-number/)

难度简单526收藏分享切换为英文接收动态反馈

编写一个算法来判断一个数 `n` 是不是快乐数。

「快乐数」定义为：

- 对于一个正整数，每一次将该数替换为它每个位置上的数字的平方和。
- 然后重复这个过程直到这个数变为 1，也可能是 **无限循环** 但始终变不到 1。
- 如果 **可以变为** 1，那么这个数就是快乐数。

如果 `n` 是快乐数就返回 `true` ；不是，则返回 `false` 。

 

**示例 1：**

```
输入：19
输出：true
解释：
12 + 92 = 82
82 + 22 = 68
62 + 82 = 100
12 + 02 + 02 = 1
```

**示例 2：**

```
输入：n = 2
输出：false
```



**题解**

* 对于看似有无限可能的，最后一定会成为循环。

```java
class Solution {
    public boolean isHappy(int n) {
        Set<Integer> record = new HashSet<>();

        while(true){
            record.add(n);

            int sum = 0;
            while(n != 0){
                sum = sum + (n % 10) * (n % 10);
                n /= 10;
            }

            if(sum == 1){
                return true;
            }
            if(record.contains(sum)){
                return false;
            }
            n = sum;
        }
    }
}
```



#### [290. 单词规律](https://leetcode-cn.com/problems/word-pattern/)

难度简单305收藏分享切换为英文接收动态反馈

给定一种规律 `pattern` 和一个字符串 `str` ，判断 `str` 是否遵循相同的规律。

这里的 **遵循** 指完全匹配，例如， `pattern` 里的每个字母和字符串 `str` 中的每个非空单词之间存在着双向连接的对应规律。

**示例1:**

```
输入: pattern = "abba", str = "dog cat cat dog"
输出: true
```

**示例 2:**

```
输入:pattern = "abba", str = "dog cat cat fish"
输出: false
```

**示例 3:**

```
输入: pattern = "aaaa", str = "dog cat cat dog"
输出: false
```

**示例 4:**

```
输入: pattern = "abba", str = "dog dog dog dog"
输出: false
```



**题解**

```java
class Solution {
    public boolean wordPattern(String pattern, String s) {
        Map<Character , String> record = new HashMap<>();
        Set<String> distinct = new HashSet<>();

        String[] arrs = s.split(" ");
        if(pattern.length() != arrs.length){
            return false;
        }

        for(int i = 0 ; i < pattern.length() ; i ++){
            if(record.get(pattern.charAt(i)) != null){
                if(! record.get(pattern.charAt(i)).equals(arrs[i])){
                    return false;
                }
            }else {
                record.put(pattern.charAt(i) , arrs[i]);
                if(distinct.contains(arrs[i])){
                    return false;
                }
                distinct.add(arrs[i]);
            }
        }

        return true;
    }
}
```



#### [205. 同构字符串](https://leetcode-cn.com/problems/isomorphic-strings/)

难度简单327收藏分享切换为英文接收动态反馈

给定两个字符串 ***s*** 和 ***t\***，判断它们是否是同构的。

如果 ***s*** 中的字符可以按某种映射关系替换得到 ***t\*** ，那么这两个字符串是同构的。

每个出现的字符都应当映射到另一个字符，同时不改变字符的顺序。不同字符不能映射到同一个字符上，相同字符只能映射到同一个字符上，字符可以映射到自己本身。

 

**示例 1:**

```
输入：s = "egg", t = "add"
输出：true
```

**示例 2：**

```
输入：s = "foo", t = "bar"
输出：false
```

**示例 3：**

```
输入：s = "paper", t = "title"
输出：true
```

 

#### [205. 同构字符串](https://leetcode-cn.com/problems/isomorphic-strings/)

难度简单327收藏分享切换为英文接收动态反馈

给定两个字符串 ***s*** 和 ***t\***，判断它们是否是同构的。

如果 ***s*** 中的字符可以按某种映射关系替换得到 ***t\*** ，那么这两个字符串是同构的。

每个出现的字符都应当映射到另一个字符，同时不改变字符的顺序。不同字符不能映射到同一个字符上，相同字符只能映射到同一个字符上，字符可以映射到自己本身。

 

**示例 1:**

```
输入：s = "egg", t = "add"
输出：true
```

**示例 2：**

```
输入：s = "foo", t = "bar"
输出：false
```

**示例 3：**

```
输入：s = "paper", t = "title"
输出：true
```

 

**题解**

```java
class Solution {
    public boolean isIsomorphic(String s, String t) {
        if(s.length() != t.length()){
            return false;
        }
        Map<Character , Character> record = new HashMap<>();
        Set<Character> distinct = new HashSet<>();

        for(int i = 0 ; i < s.length() ; i ++){
            if(record.get(t.charAt(i)) != null){
                if(record.get(t.charAt(i)) != s.charAt(i)){
                    return false;
                }
            }else {
                if(distinct.contains(s.charAt(i))){
                    return false;
                }
                record.put(t.charAt(i) , s.charAt(i));
                distinct.add(s.charAt(i));
            }
        }

        return true;
    }
}
```



#### [451. 根据字符出现频率排序](https://leetcode-cn.com/problems/sort-characters-by-frequency/)

难度中等213收藏分享切换为英文接收动态反馈

给定一个字符串，请将字符串里的字符按照出现的频率降序排列。

**示例 1:**

```
输入:
"tree"

输出:
"eert"

解释:
'e'出现两次，'r'和't'都只出现一次。
因此'e'必须出现在'r'和't'之前。此外，"eetr"也是一个有效的答案。
```

**示例 2:**

```
输入:
"cccaaa"

输出:
"cccaaa"

解释:
'c'和'a'都出现三次。此外，"aaaccc"也是有效的答案。
注意"cacaca"是不正确的，因为相同的字母必须放在一起。
```



**题解**

```java
class Solution {
    public String frequencySort(String s) {
        Map<Character , Integer> record = new HashMap<>();

        for(int i = 0 ; i < s.length() ; i ++){
            record.put(s.charAt(i) , record.getOrDefault(s.charAt(i) , 0) + 1);
        }

        PriorityQueue<Character> queue = new PriorityQueue<>((o1 , o2) -> record.get(o2) - record.get(o1));
        queue.addAll(record.keySet());

        char[] res = new char[s.length()];
        int idx = 0;
        while(queue.size() != 0){
            Character c = queue.poll();
            int times = record.get(c);
            for(int i = 0 ; i < times ; i ++){
                res[idx ++] = c;
            }
        }

        return new String(res);
    }
}
```

