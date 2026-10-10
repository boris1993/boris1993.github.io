---
title: "LeetCode刷题笔记"
source: https://www.boris1993.com/leetcode-notes.html
date: 2023-01-26
updated: 2026-10-10
tags: [LeetCode, 乐扣]
categories: [学知识]
---

# LeetCode刷题笔记

迫于找工作，不得不一边鄙视自己的智商，一边硬着头皮刷LeetCode。既然开始刷题，那顺便做个笔记，以便日后温习。

因为顺序是乱的，所以请善用`CTRL-F`。

## 217\. Contains Duplicate

> Example:
> 
> ```plaintext
> Input: nums = [1,2,3,1]
> Output: true
> 
> Input: nums = [1,2,3,4]
> Output: false
> ```

```java
class Solution {
    public boolean containsDuplicate(int[] nums) {
        if (nums == null || nums.length == 0) {
            return false;
        }

        HashSet<Integer> hashSet = new HashSet<>();
        for (int i : nums) {
            // 看见一个数字就去hashSet里面查，
            // 查到就说明重复过了
            if (hashSet.contains(i)) {
                return true;
            }

            hashSet.add(i);
        }

        return false;
    }
}
```

## 242\. Valid Anagram

字母异位词，就是两个单词长度一样，用到的字母一样，但各字母出现的位置不一样。既然如此，用一个HashMap记录两个单词各自字母出现的次数，再做一个比较，不就知道了。

```java
class Solution {
    public boolean isAnagram(String s, String t) {
        if (s.length() != t.length()) {
            return false;
        }

        // 字符串s中各个字符出现的次数
        Map<Character, Integer> countS = new HashMap<>();

        // 字符串t中各个字符出现的次数
        Map<Character, Integer> countT = new HashMap<>();

        for (int i = 0; i < s.length(); i++) {
            countS.put(
                s.charAt(i), 
                countS.getOrDefault(s.charAt(i), 0) + 1);
        }

        for (int i = 0; i < t.length(); i++) {
            countT.put(
                t.charAt(i), 
                countT.getOrDefault(t.charAt(i), 0) + 1);
        }

        // Map#equals比较的是两个map中的键值对
        // 键值对相同，说明两串字符都用了相同的字母，每个字母出现的次数也一样
        // 即同字母异序词
        return countS.equals(countT);
    }
}
```

但说起哈希，既然题目中限定了只会出现小写字母，那计算某个字母距离’a’的距离不也是一种哈希？

```java
class Solution {
    public boolean isAnagram(String s, String t) {
        // 26个字母，所以26个元素长度的数组，记录每个字母出现的次数
        final int[] charCount = new int[26];

        // 字符串s中每出现一次，记录加一
        for (char ch : s.toCharArray()) {
            charCount[ch - 'a']++;
        }

        // 字符串t中每出现一次，记录减一
        for (char ch : t.toCharArray()) {
            charCount[ch - 'a']--;
        }

        // 全部算完了之后，如果是字母异位词，
        // 那每个字母的记录加了多少次就也会减掉多少次
        for (int count : charCount) {
            if (count != 0) {
                return false;
            }
        }

        return true;
    }
}
```

## 1299\. Replace Elements with Greatest Element on Right Side

> Example:
> 
> ```plaintext
> Input: arr = [17,18,5,4,6,1]
> Output: [18,6,6,6,1,-1]
> Explanation: 
> - index 0 --> the greatest element to the right of index 0 is index 1 (18).
> - index 1 --> the greatest element to the right of index 1 is index 4 (6).
> - index 2 --> the greatest element to the right of index 2 is index 4 (6).
> - index 3 --> the greatest element to the right of index 3 is index 4 (6).
> - index 4 --> the greatest element to the right of index 4 is index 5 (1).
> - index 5 --> there are no elements to the right of index 5, so we put -1.
> ```

```java
class Solution {
    public int[] replaceElements(int[] arr) {
        int max = -1;
        for (int i = arr.length - 1; i >= 0; i--) {
            int a = arr[i];
            arr[i] = max;
            max = Math.max(max, a);
        }

        return arr;
    }
}
```

解题思路就是把explanation反过来看，既然要找元素右边最大的数，那么就从最右开始，这样找到的最大的数必定适用于`arr[i] ~ arr[length - 1]`这个范围。

## 392\. Is Subsequence

> Example:
> 
> ```plaintext
> Input: s = "abc", t = "ahbgdc"
> Output: true
> 
> Input: s = "axc", t = "ahbgdc"
> Output: false
> 
> Input: s = "acb", t = "ahbgdc"
> Output: false
> 
> Input: s = "aaaaaa", t = "bbaaaa"
> Output: false
> ```

```java
class Solution {
    public boolean isSubsequence(String s, String t) {
        if (s.equals(t) || s.length() == 0) {
            return true;
        }

        if (t.length() == 0) {
            return false;
        }

        int sIndex = 0;
        for (int i = 0; i < t.length(); i++) {
            if (t.charAt(i) == s.charAt(sIndex)) {
                sIndex++;
                if (sIndex >= s.length()) {
                    return true;
                }
            }
        }

        return false;
    }
}
```

解题思路就是整两个指针，`sIndex`指向`s`的各个字符，在循环里面逐个取`t`的字符跟`s[sIndex]`对比，匹配到的话`sIndex`就往下走一步，如果`sIndex`能走到头，就说明`s`是`t`的子序列。

## 58\. Length of Last Word

> Example:
> 
> ```plaintext
> Input: s = "Hello World"
> Output: 5
> Explanation: The last word is "World" with length 5.
> 
> Input: s = "   fly me   to   the moon  "
> Output: 4
> Explanation: The last word is "moon" with length 4.
> ```

```java
class Solution {
    public int lengthOfLastWord(String s) {
        String[] strs = s.split(" ");
        return strs[strs.length - 1].length();
    }
}
```

我的评价是，这道题不应该出现在LeetCode，应该出现在大学Java课程的作业里。

## 14\. Longest Common Prefix

> Example:
> 
> ```plaintext
> Input: strs = ["flower","flow","flight"]
> Output: "fl"
> ```

```java
class Solution {
    public String longestCommonPrefix(String[] strs) {
        if (strs.length == 0) {
            return "";
        }

        // 拿第一个字符串当模版，匹配剩下几个字符串
        String commonPrefix = strs[0];
        for (int i = 1; i < strs.length; i++) {
            // 如果commonPrefix在strs[i]的下标不是0
            // 那么就去掉commonPrefix的最后一个字母，直到下标为0
            // 如果没有common prefix，那么commonPrefix会被砍成空字符串
            while (strs[i].indexOf(commonPrefix) != 0) {
                commonPrefix = commonPrefix.substring(0, commonPrefix.length() - 1);
            }
        }

        return commonPrefix;
    }
}
```

计算过程：（语言描述太费劲，直接拿Replit放示意图算了）

![](https://blog-static.boris1993.com/leetcode-notes/longest-common-prefix.png)

## 49\. Group Anagrams

> Example:
> 
> ```plaintext
> Input: strs = ["eat","tea","tan","ate","nat","bat"]
> Output: [["bat"],["nat","tan"],["ate","eat","tea"]]
> ```

```java
class Solution {
    public List<List<String>> groupAnagrams(String[] strs) {
        Map<String, List<String>> map = new LinkedHashMap<>();

        for (String str: strs) {
            // 取一个字符串，打散成字符数组，把字符数组排序后得到一个新字符串
            // 比如eat-> aet，tea->aet
            // 这个作为map的key
            char[] charArray = str.toCharArray();
            Arrays.sort(charArray);
            String newStr = new String(charArray);

            if (!map.containsKey(newStr)) {
                map.put(newStr, new ArrayList<>());
            }
            // 把重排序后结果相同的字符串放在同一个key下面的List里面
            map.get(newStr).add(str);
        }

        List<List<String>> result = new ArrayList(map.values());
        return result;
    }
}
```

计算过程语言不好描述……但是代码挺易懂的吧，实在看不明白的话自己debug一下就清楚了。

## 118\. Pascal’s Triangle

> Example:
> 
> ```plaintext
> Input: numRows = 5
> Output: [[1],[1,1],[1,2,1],[1,3,3,1],[1,4,6,4,1]]
> 
> Input: numRows = 1
> Output: [[1]]
> ```
> 
> ![](https://upload.wikimedia.org/wikipedia/commons/0/0d/PascalTriangleAnimated2.gif)

```java
class Solution {
    public List<List<Integer>> generate(int numRows) {
        // 需要单独处理只有一行的情况
        // 否则会因为下面预设前两行数据的代码而返回错误的数据
        if (numRows == 1) {
            return List.of(List.of(1));
        }

        List<List<Integer>> result = new ArrayList<>(numRows);
        
        // 前两行的内容是固定的
        result.add(List.of(1));
        result.add(List.of(1,1));

        for (int row = 2; row < numRows; row++) {
            List<Integer> list = new ArrayList<>(row + 1);
            // 这一行最左边肯定是1
            list.add(1);

            // 两个两个取上一行的各个数，两两相加，得到当前格子的数字
            // 因为要取上一行的第i和第i+1个元素，所以循环结束条件得是上一行的个数减一
            // 否则就下标越界了
            int previousRow = row - 1;
            for (int i = 0; i < result.get(previousRow).size() - 1; i++) {
                int sum = result.get(previousRow).get(i) + result.get(previousRow).get(i + 1);
                list.add(sum);
            }
            
            // 最右边也肯定是1
            list.add(1);

            result.add(list);
        }

        return result;
    }
}
```

## 27\. Remove Element

> Example:
> 
> ```plaintext
> Input: nums = [3,2,2,3], val = 3
> Output: 2, nums = [2,2,_,_]
> Explanation: Your function should return k = 2, with the first two elements of nums being 2.
> It does not matter what you leave beyond the returned k (hence they are underscores).
> ```

```java
class Solution {
    public int removeElement(int[] nums, int val) {
        // 不等于val的数字的个数，同时当作nums的一个指针
        int count = 0;
        for (int i = 0; i < nums.length; i++) {
            if (nums[i] != val) {
                nums[count] = nums[i];
                count++;
            }
        }

        return count;
    }
}
```

## 929\. Unique Email Addresses

> Example:
> 
> ```plaintext
> Input: emails = ["test.email+alex@leetcode.com","test.e.mail+bob.cathy@leetcode.com","testemail+david@lee.tcode.com"]
> Output: 2
> Explanation: "testemail@leetcode.com" and "testemail@lee.tcode.com" actually receive mails.
> ```

```java
class Solution {
    public int numUniqueEmails(String[] emails) {
        HashSet<String> uniqueEmails = new HashSet<>();

        for (String email : emails) {
            String sanitizedEmail = "";
            int indexOfPlusSign = email.indexOf("+");
            int indexOfAtSign = email.indexOf("@");

            if (indexOfPlusSign > 0) {
                // 按题目要求，第一个加号及其以后的用户名部分都会被忽略
                sanitizedEmail += email.substring(0, indexOfPlusSign);
            } else {
                sanitizedEmail += email.substring(0, indexOfAtSign);
            }

            // 按题目要求，用户名部分的点都会被忽略
            sanitizedEmail = sanitizedEmail.replace(".", "");

            // 处理完了用户名部分，把域名部分拼上去
            sanitizedEmail += email.substring(indexOfAtSign);

            // 全处理完之后，扔进HashSet里面，顺便去重
            uniqueEmails.add(sanitizedEmail);
        }

        return uniqueEmails.size();
    }
}
```

## 205\. Isomorphic Strings

> Example:
> 
> ```plaintext
> Input: s = "paper", t = "title"
> Output: true
> 
> Input: s = "foo", t = "bar"
> Output: false
> ```

```java
class Solution {
    public boolean isIsomorphic(String s, String t) {
        // 俩字符串长度都不一样，那必然不同形
        if (s.length() != t.length()) {
            return false;
        }

        // 建立s中每个字符与t中同位置字符的映射关系
        // 如 egg -> add
        // e -> a, g -> d
        HashMap<Character, Character> map = new HashMap(s.length());
        for (int i = 0; i < t.length(); i++) {
            if (map.containsValue(t.charAt(i))) {
                continue;
            }

            map.put(s.charAt(i), t.charAt(i));
        }

        // 然后从map里面，按照s的每个字母，取出映射过的字符
        // 拼在StringBuilder里面
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < s.length(); i++) {
            Character ch = map.get(s.charAt(i));
            sb.append(ch);
        }

        // 检查用映射拼出来的字符串与t是否相同
        return sb.toString().equals(t);
    }
}
```

## 347\. Top K Frequent Elements

> Example:
> 
> ```plaintext
> Input: nums = [1,1,1,2,2,3], k = 2
> Output: [1,2]
> ```

```java
class Solution {
    public int[] topKFrequent(int[] nums, int k) {
        if (nums.length == k) {
            return nums;
        }

        return Arrays
            .stream(nums)
            // int -> Integer
            .boxed()
            // 收集到一个map中，key是数字，value是出现的次数
            .collect(Collectors.groupingBy(num -> num, Collectors.summingInt(num -> 1)))
            .entrySet()
            .stream()
            // 把entry set按照value降序排列
            .sorted(Map.Entry.comparingByValue((a, b) -> Integer.compare(b, a)))
            // 取前k个
            .limit(k)
            // 把key取出来，unbox成int
            .mapToInt(e -> e.getKey())
            // 最后造个数组出来返回掉
            .toArray();
    }
}
```

还写(chao)了一个不用`stream`，纯手工拿entry set做比较的解法，因过于丑陋，就不贴在这了，`submission`在这里。

## 128\. Longest Consecutive Sequence

> Example:
> 
> ```plaintext
> Input: nums = [100,4,200,1,3,2]
> Output: 4
> Explanation: The longest consecutive elements sequence is [1, 2, 3, 4]. Therefore its length is 4.
> 
> Input: nums = [0,3,7,2,5,8,4,6,0,1]
> Output: 9
> Explanation: The longest consecutive elements sequence is [0, 1, 2, 3, 4, 5, 6, 7, 8]. Therefore its length is 9.
> ```

```java
class Solution {
    public int longestConsecutive(int[] nums) {
        if (nums.length == 0) {
            return 0;
        }

        Arrays.sort(nums);

        int longestStreak = 1;
        int currentStreak = 1;

        for (int i = 1; i < nums.length; i++) {
            // 因为当前数字已经记入了一个“连击”
            // 所以当这个数字跟上一个数字重复的时候，直接跳到下一个数字
            if (nums[i] != nums[i - 1]) {
                // 既然连续，那么当前数字跟上一个数字肯定差1
                if (nums[i] == nums[i - 1] + 1) {
                    currentStreak++;
                } else {
                    // 如果不连续了，记下来最大的连击数，当前连击数重置
                    longestStreak = Math.max(longestStreak, currentStreak);
                    currentStreak = 1;
                }
            }
        }

        // 最后取最大的连击数，这个不用多说吧
        return Math.max(longestStreak, currentStreak);
    }
}
```

## 125\. Valid Palindrome

> Example:
> 
> ```plaintext
> Input: s = "A man, a plan, a canal: Panama"
> Output: true
> Explanation: "amanaplanacanalpanama" is a palindrome.
> 
> Input: s = " "
> Output: true
> Explanation: s is an empty string "" after removing non-alphanumeric characters.
> Since an empty string reads the same forward and backward, it is a palindrome.
> ```

```java
class Solution {
    public boolean isPalindrome(String s) {
        // 把不是字母和数字的字符剔出去，然后转小写，方便比较
        String str = s.replaceAll("[\\W]|_", "");
        str = str.toLowerCase();

        String reversed = new StringBuilder(str).reverse().toString();

        return str.equals(reversed);
    }
}
```

但是很显然，这个偷鸡解法并不是`two pointers`这个分类想要的，所以另一个解法：

```java
class Solution {
    public boolean isPalindrome(String s) {
        // 把不是字母和数字的字符剔出去，然后转小写，方便比较
        String str = s.replaceAll("[\\W]|_", "");
        str = str.toLowerCase();

        char[] chars = str.toCharArray();

        int middle = chars.length / 2;
        int head = 0;
        int tail = chars.length - 1;

        // head和tail两个指针逐步向middle逼近
        // 一边走，一边比较两个指针指向的字母是不是一样
        while (head < middle) {
            if (chars[head] != chars[tail]) {
                // 不一样的话，那自然就不是回文字符串了
                return false;
            }

            head++;
            tail--;
        }

        return true;
    }
}
```

## 1\. Two Sum

> Example:
> 
> ```plaintext
> Input: nums = [2,7,11,15], target = 9
> Output: [0,1]
> Explanation: Because nums[0] + nums[1] == 9, we return [0, 1].
> ```

```java
class Solution {
    public int[] twoSum(int[] nums, int target) {
        // key是数字本身，value是数字所在下标
        Map<Integer, Integer> map = new HashMap<>();
        
        for (int i = 0; i < nums.length; i++) {
            // 算一下target与当前下标的差
            int complement = target - nums[i];
            if (map.containsKey(complement)) {
                return new int[] { map.get(complement), i };
            }
            map.put(nums[i], i);
        }
        
        throw new IllegalArgumentException("No solution found");
    }
}
```

计算过程：

-   `i = 0`，`nums[i] = 2`，`complement = 9 - 2 = 7`，map的key里面找不到7，所以`map.put(2, 0)`
-   `i = 1`，`nums[i] = 7`，`complement = 9 - 7 = 2`，map的key里面有2，即`nums[i] + nums[map.get(2)] = 9`，返回`[map.get(2), nums[i]]`即`[0, 1]`

## 167\. Two Sum II - Input Array Is Sorted

> Example:
> 
> ```plaintext
> Input: numbers = [2,7,11,15], target = 9
> Output: [1,2]
> Explanation: The sum of 2 and 7 is 9. Therefore, index1 = 1, index2 = 2. We return [1, 2].
> ```

```java
class Solution {
    public int[] twoSum(int[] numbers, int target) {
        // 两个指针
        // 一个从头往尾走，一个从尾往头走
        int low = 0;
        int high = numbers.length - 1;

        while (low < high) {
            // 要求是计算两个数字之和是否等于target
            // 同时因为数组已经按升序排列
            // 那么如果加的结果大于target，就让尾指针往前，取更小的数
            // 如果结果小于target，就让头指针往后，取更大的数
            // 结果要么找到合适的两个数，要么两个指针相遇，即没有合适的数字
            int sum = numbers[low] + numbers[high];
            if (sum == target) {
                return new int[] {low + 1, high + 1};
            } else if (sum < target) {
                ++low;
            } else {
                --high;
            }
        }

        return new int[] {-1, -1};
    }
}
```

## 15\. 3Sum

> Example:
> 
> ```plaintext
> Input: nums = [-1,0,1,2,-1,-4]
> Output: [[-1,-1,2],[-1,0,1]]
> Explanation: 
> nums[0] + nums[1] + nums[2] = (-1) + 0 + 1 = 0.
> nums[1] + nums[2] + nums[4] = 0 + 1 + (-1) = 0.
> nums[0] + nums[3] + nums[4] = (-1) + 2 + (-1) = 0.
> The distinct triplets are [-1,0,1] and [-1,-1,2].
> Notice that the order of the output and the order of the triplets does not matter.
> ```

一种解法是，类似`Two Sum II`，取一个基准数，然后用`Two Sum II`的方法找基准数右边符合要求的两个数。

```java
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {
        // 先把nums排序，让这个数组符合Two Sum II的要求
        Arrays.sort(nums);

        List<List<Integer>> result = new ArrayList<>();
        
        // 从头到尾遍历nums，取一个数字作为基准
        for (int i = 0; i < nums.length && nums[i] <= 0; ++i) {
            // 这里的条件是为了不重复计算相同的数字，
            // 即只在当前数字不跟上一个数字重复的时候才计算
            if (i == 0 || nums[i - 1] != nums[i]) {
                twoSumII(nums, i, result);
            }
        }

        return result;
    }

    private void twoSumII(int[] nums, int i, List<List<Integer>> result) {
        // 取nums[i]右边的数组作为子数组
        // 左指针指向子数组第一个元素
        int low = i + 1;
        // 右指针指向最后一个元素
        int high = nums.length - 1;

        while (low < high) {
            // 基准数 + 左指针 + 右指针
            int sum = nums[i] + nums[low] + nums[high];
            if (sum < 0) {
                // 结果小于零，那么左指针往右走，取更大的一个数
                ++low;
            } else if (sum > 0) {
                // 结果大于零，那么右指针往左走，取更小的一个数
                --high;
            } else {
                // 等于零，那么这三个数就是我们想要的
                result.add(Arrays.asList(nums[i], nums[low], nums[high]));
                
                // 继续缩小范围
                low++;
                high--;

                // 并且当low指向相同数字时，继续向右走
                // 然后重新在上一层循环里面继续找符合要求的数字
                // 毕竟区间内可能有多组符合条件的数字
                while (low < high && nums[low] == nums[low - 1]) {
                    ++low;
                }
            }
        }
    }
}
```

但是，如果题目不允许改变`nums`数组呢？那么可以这样解，

```java
class Solution {
    public List<List<Integer>> threeSum(int[] nums) {
        Set<List<Integer>> result = new HashSet<>();
        Set<Integer> duplicates = new HashSet<>();
        Map<Integer, Integer> seen = new HashMap<>();

        for (int i = 0; i < nums.length; i++) {
            if (!duplicates.add(nums[i])) {
                continue;
            }

            for (int j = i + 1; j < nums.length; j++) {
                int complement = -(nums[i] + nums[j]);
                // 如果在seen这个map里面找到了曾经计算过的complement
                // 那么说明complement存在于nums数组中
                // 即找到了三个和为零的数字，nums[i]，nums[j]，complement
                if (seen.containsKey(complement) && seen.get(complement) == i) {
                    List<Integer> triplet = Arrays.asList(nums[i], nums[j], complement);
                    // 排序之后再放进set，避免插入内容相同位置不同的结果
                    Collections.sort(triplet);
                    result.add(triplet);
                }

                // 记录nums[j]可以作为nums[i]的补充（complement）
                seen.put(nums[j], i);
            }
        }

        return new ArrayList<>(result);
    }
}
```

## 11\. Container With Most Water

> Example:
> 
> ```plaintext
> Input: height = [1,8,6,2,5,4,8,3,7]
> Output: 49
> Explanation: The above vertical lines are represented by array [1,8,6,2,5,4,8,3,7]. In this case, the max area of water (blue section) the container can contain is 49.
> ```
> 
> ![](https://s3-lc-upload.s3.amazonaws.com/uploads/2018/07/17/question_11.jpg)

```java
class Solution {
    public int maxArea(int[] height) {
        int max = 0;

        int left = 0; 
        int right = height.length - 1;

        while (left < right) {
            // 取容器最短的那个板子，木桶原理嘛
            int ht = Math.min(height[left], height[right]);
            // 板子左右有几格水
            int water = right - left;
            // 水的体积，也就是蓝色正方形的面积，底*高
            int volume = ht * water;
            // 记录最大的体积
            max = Math.max(max, volume);

            // 既然要找最大的体积，当然哪个板子短就换哪个
            if (height[left] < height[right]) {
                left++;
            } else {
                right--;
            }
        }

        return max;
    }
}
```

## 121\. Best Time to Buy and Sell Stock

> Example:
> 
> ```plaintext
> Input: prices = [7,1,5,3,6,4]
> Output: 5
> Explanation: Buy on day 2 (price = 1) and sell on day 5 (price = 6), profit = 6-1 = 5.
> Note that buying on day 2 and selling on day 1 is not allowed because you must buy before you sell.
> ```

```java
class Solution {
    public int maxProfit(int[] prices) {
        if (prices.length < 2) {
            return 0;
        }

        int buyPrice = Integer.MAX_VALUE;
        int overallProfit = 0;
        int profitIfSoldToday = 0;

        for (int i = 0; i < prices.length; i++) {
            // 让buyPrice保存最小的价格
            if (prices[i] < buyPrice) {
                buyPrice = prices[i];
            }

            // 计算今日价格与买入价的差
            profitIfSoldToday = prices[i] - buyPrice;

            // overallProfit记录最大的总利润
            if (overallProfit < profitIfSoldToday) {
                overallProfit = profitIfSoldToday;
            }
        }

        return overallProfit;
    }
}
```

## 108\. Convert Sorted Array to Binary Search Tree

> Example:
> 
> ```plaintext
> Input: nums = [-10,-3,0,5,9]
> Output: [0,-3,9,-10,null,5]
> ```
> 
> ![](https://assets.leetcode.com/uploads/2021/02/18/btree1.jpg)

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode() {}
 *     TreeNode(int val) { this.val = val; }
 *     TreeNode(int val, TreeNode left, TreeNode right) {
 *         this.val = val;
 *         this.left = left;
 *         this.right = right;
 *     }
 * }
 */
class Solution {
    public TreeNode sortedArrayToBST(int[] nums) {
        return builder(nums, 0, nums.length - 1);
    }
    
    private TreeNode builder(int[] nums, int left, int right) {
        if (left > right) {
            return null;
        }

        // 从中间将nums二分
        // 比如在第一层递归中，
        // [-10,-3,0,5,9] -> [-10,-3,0] [5,9]
        int middle = (left + right) / 2;

        // 中间的数字作为二叉搜索树的根节点
        TreeNode rootNode = new TreeNode(nums[middle]);
        // 第一层递归中[-10, -3]拿去构造左边的子节点
        rootNode.left = builder(nums, left, middle - 1);
        // 第一层递归中[5,9]拿去构造右边的子节点
        rootNode.right = builder(nums, middle + 1, right);

        return rootNode;
    }
}
```

思路就是递归+二分，或许还有些分治思想？先找见根节点，然后把数组左右分成两半，在递归里面再重复这样的操作，直到只有一个根节点，也就是最下面的叶子节点。最后往上组装。  
数据怎么跑的debug一下看看吧，用语言描述肯定要乱死。

## 146\. LRU Cache

> Example:
> 
> ```plaintext
> Input
> ["LRUCache", "put", "put", "get", "put", "get", "put", "get", "get", "get"]
> [[2], [1, 1], [2, 2], [1], [3, 3], [2], [4, 4], [1], [3], [4]]
> Output
> [null, null, null, 1, null, -1, null, -1, 3, 4]
> 
> Explanation
> LRUCache lRUCache = new LRUCache(2);
> lRUCache.put(1, 1); // cache is {1=1}
> lRUCache.put(2, 2); // cache is {1=1, 2=2}
> lRUCache.get(1);    // return 1
> lRUCache.put(3, 3); // LRU key was 2, evicts key 2, cache is {1=1, 3=3}
> lRUCache.get(2);    // returns -1 (not found)
> lRUCache.put(4, 4); // LRU key was 1, evicts key 1, cache is {4=4, 3=3}
> lRUCache.get(1);    // return -1 (not found)
> lRUCache.get(3);    // return 3
> lRUCache.get(4);    // return 4
> ```

解题思路：

LRU Cache，即`Least Recently Used Cache`，其运作机理是，如果在cache已满的时候添加新的记录，那么要先删掉其中最不常用的记录，然后添加新的记录。

我们可以用双向链表来实现这个效果，越靠近链表头，就代表这个元素越常被用到；反之越靠近链表尾，这个元素就越不常用，链表尾的前一个元素也将是在cache满了之后被删掉的那个元素。

那么首先需要创建一个类`Node`，代表双向链表中的节点。

```java
public class Node {
    Node next;
    Node prev;

    int key;
    int value;

    public Node(int key, int value) {
        this.key = key;
        this.value = value;
    }
}
```

这时候就可以初始化`LRUCache`这个类的结构了。

```java
class LRUCache {
    // 链表头节点，永远不动
    private final Node head = new Node(0, 0);
    // 链表尾节点，永远不动
    private final Node tail = new Node(0, 0);
    // 存储key与Node的映射，get的时候实际上是从这取值
    private final Map<Integer, Node> map = new HashMap<>();
    // cache容量
    private final int capacity;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        // 双向链表头尾相连
        head.next = tail;
        tail.prev = head;
    }
}
```

接下来先实现`get`方法：

```java
// 当一个节点被get
// 那么就把它提到双向链表的头部位置
public int get(int key) {
    // 根据题目要求，key不存在就返回-1
    if (!map.containsKey(key)) {
        return -1;
    }

    Node node = map.get(key);
    Node nodeNext = node.next;
    Node nodePrev = node.prev;

    // 先把这个node从当前位置抹去
    nodePrev.next = nodeNext;
    nodeNext.prev = nodePrev;

    // 然后插入到头部
    Node headNext = head.next;
    node.next = headNext;
    node.prev = head;
    headNext.prev = node;
    head.next = node;

    return node.value;
}
```

接下来实现`put`方法：

```java
public void put(int key, int value) {
    // 如果key已经存在
    // 那么put相当于更新value，同时刷新在LRU Cache中的位置
    // 所以先把已有的这个node删掉
    if (map.containsKey(key)) {
        Node node = map.remove(key);
        
        Node nodeNext = node.next;
        Node nodePrev = node.prev;
        nodePrev.next = nodeNext;
        nodeNext.prev = nodePrev;
    }

    // 如果cache满了，就要先删掉末尾的节点（末位淘汰制？
    if (map.size() == capacity) {
        Node leastRecentUsedNode = tail.prev;
        map.remove(leastRecentUsedNode.key);

        leastRecentUsedNode.prev = tail;
        tail.prev = leastRecentUsedNode.prev;
    }

    Node newNode = new Node(key, value);
    map.put(key, newNode);

    // 把这个新建的node塞到head和headNext之间
    Node headNext = head.next;
    newNode.prev = head;
    newNode.next = headNext;
    headNext.prev = newNode;
    head.next = newNode;
}
```

可见新增节点和删除节点操作的代码是经常重复的，所以抽成两个单独的方法。最后完整的解题代码如下：

```java
class LRUCache {
    private final Node head = new Node(0, 0);
    private final Node tail = new Node(0, 0);
    private final Map<Integer, Node> map = new HashMap<>();

    private final int capacity;
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }
    
    public int get(int key) {
        if (!map.containsKey(key)) {
            return -1;
        }

        Node node = map.get(key);
        remove(node);
        insert(node);

        return node.value;
    }
    
    public void put(int key, int value) {
        if (map.containsKey(key)) {
            remove(map.get(key));
        }

        if (map.size() == capacity) {
            remove(tail.prev);
        }

        insert(new Node(key, value));
    }

    private void remove(Node node) {
        map.remove(node.key);
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void insert(Node node) {
        map.put(node.key, node);

        Node headNext = head.next;
        node.prev = head;
        node.next = headNext;
        headNext.prev = node;
        head.next = node;
    }

    public class Node {
        Node next;
        Node prev;

        int key;
        int value;

        public Node(int key, int value) {
            this.key = key;
            this.value = value;
        }
    }
}

/**
 * Your LRUCache object will be instantiated and called as such:
 * LRUCache obj = new LRUCache(capacity);
 * int param_1 = obj.get(key);
 * obj.put(key,value);
 */
```

## 9\. Palindrome number

> Example:
> 
> ```plaintext
> Input: x = 121
> Output: true
> Explanation: 121 reads as 121 from left to right and from right to left.
> ```

```java
class Solution {
    public boolean isPalindrome(int x) {
        if (x < 0) {
            return false;
        }
        
        int input = x;
        
        int reversedNum = 0;
        
        // x从低位往高位取数字，然后从高位到低位填给reversedNum
        // 即把x反转
        while (x != 0) {
            // reversedNum乘10，加上x除以10的余数，即当前x的末位数
            reversedNum = reversedNum * 10 + x % 10;
            // x除以10，抛掉当前的末位数
            x = x / 10;
        }
    
        if (reversedNum > Integer.MAX_VALUE || reversedNum < Integer.MIN_VALUE) {
            return false;
        }
        
        return input == reversedNum;
    }
}
```

计算过程：

-   `x = 121`，`reversedNum = 0`；`reversedNum = reversedNum * 10 + x % 10 = 0 + 1 = 1`；`x = x / 10 = 121 / 10 = 12`
-   `x = 12`，`reversedNum = 1`；`reversedNum = reversedNum * 10 + x % 10 = 10 + 2 = 12`；`x = x / 10 = 12 / 10 = 1`
-   `x = 1`，`reversedNum = 12`；`reversedNum = reversedNum * 10 + x % 10 = 120 + 1 = 121`；`x = x / 10 = 1 / 10 = 0`
-   `input == reversedNum => 121 == 121 => true`

写到这想到还有个粗暴解法，把数字当成字符串，翻转一下然后比较两个字符串是不是一样不就行了，做什么数学题？

```java
class Solution {
    public boolean isPalindrome(int x) {
        String inputString = Integer.toString(x);
        String reversedString = new StringBuilder(inputString).reverse().toString();

        return inputString.equals(reversedString);
    }
}
```

多清爽，三行完事还不烧脑子。

## 13\. Roman to Integer

> Example:
> 
> ```plaintext
> Input: s = "MCMXCIV"
> Output: 1994
> Explanation: M = 1000, CM = 900, XC = 90 and IV = 4.
> ```

```java
class Solution {
    public int romanToInt(String s) {
        // 基本罗马数字
        final HashMap<Character, Integer> map = new HashMap<>(7);
        map.put('I', 1);
        map.put('V', 5);
        map.put('X', 10);
        map.put('L', 50);
        map.put('C', 100);
        map.put('D', 500);
        map.put('M', 1000);

        // IV这种左减格式的数字收拾起来太麻烦，
        // 直接转成连续的基本数字
        final String sanitized = s
                .replace("IV", "IIII")
                .replace("IX", "VIIII")
                .replace("XL", "XXXX")
                .replace("XC", "LXXXX")
                .replace("CD", "CCCC")
                .replace("CM", "DCCCC");

        final char[] chars = sanitized.toCharArray();

        int result = 0;
        for (char aChar : chars) {
            result += map.get(aChar);
        }

        return result;
    }
}
```

计算过程：

-   `MCMXCIV`(1994) => `M CM XC IV`(1000 900 90 4) => `M DCCC LXXXX IIII`
-   从头到尾遍历`MDCCCLXXXXIIII`中每个字符，去map里面找对应的阿拉伯数字，加起来就完事了

一开始做的时候愁死我了，没有左减格式的数字没啥难度，想破脑袋也没想到怎么处理左减，抄答案发现还能这么玩，属实打开思路了。

## 252\. Meeting Rooms

> Given an array of meeting time `intervals` where `intervals[i] = [starti, endi]`, determine if a person could attend all meetings.
> 
> Example:
> 
> ```plaintext
> Input: intervals = [[0,30],[5,10],[15,20]]
> Output: false
> ```

Easy级别的题，思路很简单，先把会议安排按照开始时间升序排列，然后比较下一场会议的开始时间是否小于上一场会议的结束时间，是的话就说明这个人无法参加全部会议。

```java
class Solution {
    public boolean canAttendMeetings(int[][] intervals) {
        if (intervals.length <= 1) {
            return true;
        }

        // Sort with the meeting start time
        Arrays.sort(intervals, (a, b) -> a[0] - b[0]);

        int lastEndTime = intervals[0][1];

        for (int i = 1; i < intervals.length; i++) {
            int currentBeginTime = intervals[i][0];

            if (currentBeginTime < lastEndTime) {
                return false;
            }

            lastEndTime = intervals[i][1];
        }

        return true;
    }
}
```

## 253\. Meeting Rooms II

> Given an array of meeting time intervals `intervals` where `intervals[i] = [starti, endi]`, return the minimum number of conference rooms required.
> 
> Example:
> 
> ```plaintext
> Input: intervals = [[0,30],[5,10],[15,20]]
> Output: 2
> ```

这道题要用到优先队列，具体算法见注释。

```java
class Solution {
    public int minMeetingRooms(int[][] intervals) {
        if (intervals.length <= 1) {
            return intervals.length;
        }

        // 定义一个优先队列，存储会议的结束时间，按升序排列
        // 因为会议结束，下一场会议才能占用这个会议室
        // 所以只需要比较当前会议的开始时间与上一场会议的结束时间
        PriorityQueue<Integer> allocator = new PriorityQueue<>(
            intervals.length,
            (a, b) -> a - b
        );

        // 重新排序intervals，按照会议的开始时间升序排列
        Arrays.sort(
            intervals, 
            (a, b) -> a[0] - b[0]);

        // 第一场会议占用一个会议室
        allocator.add(intervals[0][1]);

        for (int i = 1; i < intervals.length; i++) {
            // 从第二场会议开始检查
            if (intervals[i][0] >= allocator.peek()) {
                // 如果第二场会议在开始的时候，上一场会议已经结束
                // 那么释放掉上一场会议的会议室
                // 也就是删除掉优先队列的第一个元素
                allocator.poll();
            }

            // 当前会议肯定要占一个会议室
            allocator.add(intervals[i][1]);
        }

        return allocator.size();
    }
}
```

## 655\. Print Binary Tree

抄答案Java Easy Solution - shawonnirob16

```java
/**
 * Definition for a binary tree node.
 * public class TreeNode {
 *     int val;
 *     TreeNode left;
 *     TreeNode right;
 *     TreeNode() {}
 *     TreeNode(int val) { this.val = val; }
 *     TreeNode(int val, TreeNode left, TreeNode right) {
 *         this.val = val;
 *         this.left = left;
 *         this.right = right;
 *     }
 * }
 */
class Solution {
    public List<List<String>> printTree(TreeNode root) {
        List<List<String>> result = new ArrayList<>();

        int height = getHeight(root);
        int row = height + 1;
        int column = (int) Math.pow(2, height + 1) - 1;

        // 先全部填充空字符串
        for (int k = 0; k < row; k++) {
            List<String> list = new ArrayList<>();
            for (int i = 0; i < column; i++) {
                list.add("");
            }

            result.add(list);
        }

        int left = 0;
        int right = column - 1;
        int level = 0;
        fill(result, left, right, level, root);

        return result;
    }

    // 递归整个树，每层高度加一，计算树的高度
    private int getHeight(TreeNode root) {
        if (root == null) {
            return -1;
        }

        int left = getHeight(root.left);
        int right = getHeight(root.right);

        return 1 + Math.max(left, right);
    }

    private void fill(
        List<List<String>> result,
        int left,
        int right,
        int level,
        TreeNode root
    ) {
        if (root == null) {
            return;
        }

        // 计算list的中间位置的下标
        int middle = left + (right - left) / 2;
        // 在中间位置填入当前节点的值
        result.get(level).set(middle, String.valueOf(root.val));

        fill(result, left, middle - 1, level + 1, root.left);
        fill(result, middle + 1, right, level + 1, root.right);
    }
}
```

## 54\. Spiral Matrix

按题目要求，我们要沿着矩阵的边缘绕圈。既然是沿着边缘，那么我们就可以分别挤压四个边（按照左上、右上、右下、左下的顺序，就是逐步压缩上、右、下、左四个边）。

```java
class Solution {
    public List<Interger> spiralOrder(int[][] matrix) {
        final List<Integer> list = new ArrayList<>();
        int top = 0;
        int bottom = matrix.length - 1;
        int left = 0;
        int right = matrix[0].length - 1;

        // 横向边界最终会被挤压到最中间一行，纵向边界同理，所以用小于等于
        while (top <= bottom && left <= right) {
            for (int y = left; y <= right; y++) {
                list.add(matrix[top][y]);
            }
            // 遍历完最顶上一行后，就收缩上边界
            // 下面处理下、左、右边界同理
            top++;

            for (int x = top; x <= bottom; x++;) {
                list.add(matrix[x][right]);
            }
            right--;

            // 在只剩下最后一行时，上面已经处理完毕，top > bottom
            // 此时如果不加以限定，那么遍历时又会从这最后一行自右向左重复添加数据
            if (top <= bottom) {
                for (int y = right; y >= left; y--) {
                    list.add(matrix[bottom][y]);
                }
                bottom--;
            }
            
            // 同上，在只剩下[[1],[2],[3]]这样的最后一列后，left > right
            // 不加以限定的话，又会自下而上重复添加
            if (left <= right) {
                for (int x = bottom; x >= top; x--) {
                    list.add(matrix[left][x]);
                }
                left++;
            }
        }

        return list;
    }
}
```

## 59\. Spiral Matrix II

跟[54\. Spiral Matrix](#54-Spiral-Matrix)一模一样的套路。

```java
class Solution {
    public int[][] generateMatrix(int n) {
        final int[][] matrix = new int[n][n];
        int top = 0;
        int bottom = n - 1;
        int left = 0;
        int right = n - 1;
        int count = 1;

        while (left <= right && top <= bottom) {
            for (int y = left; y <= right; y++) {
                matrix[top][y] = count;
                count++;
            }
            top++;

            for (int x = top; x <= bottom; x++) {
                matrix[x][right] = count;
                count++;
            }
            right--;

            if (top <= bottom) {
                for (int y = right; y >= left; y--) {
                    matrix[bottom][y] = count;
                    count++;
                }
                bottom--;
            }

            if (left <= right) {
                for (int x = bottom; x >= top; x--) {
                    matrix[x][left] = count;
                    count++;
                }
                left++;
            }
        }

        return matrix;
    }
}
```

## 203\. Remove Linked List Elements

从链表中删掉指定元素，并返回处理后的链表的头。

首先需要确定新链表的头的位置，因为可能会处理`head = [7, 7, 1, 2, 3], val = 7`这种情况。然后类似双指针，一个从新的链表头出发（current），另一个从新链表头的下一个元素出发（next），遍历链表，遇到`next.val == val`就重新指定`current.next`以删除元素。

```java
class Solution {
    public ListNode removeElements(ListNode head, int val) {
        // Skip to the first node whose value is not the target value
        // e.g.:
        // 2->2->3->1->2->4, target = 2
        // becomes
        // 2->3->1->2->4
        // then
        // 3->1->2->4
        while (head != null && head.val == val) {
            head = head.next;
        }

        // Just in case a list node is 
        // 2->2->2->2
        // and the target value is 2
        if (head == null) {
            return head;
        }

        ListNode current = head;
        ListNode next = head.next;
        while (next != null) {
            if (next.val == val) {
                current.next = next.next;
            } else {
                current = next;
            }
            next = next.next;
        }

        return head;
    }
}
```

## 707\. Design Linked List

这题实际上是要设计一个类，类的成员里面有一个链表，以及实现几个操作这个链表的方法。

折腾链表，推荐创建一个虚拟的头节点head，真实的头节点从head.next开始。这样我们所有的操作都能从一个保证不是null的对象开始，用一点点空间换操作的省心。

```java
class MyLinkedList {
    class ListNode {
        int val;
        ListNode next;
        
        public ListNode(int val) {
            this.val = val;
        }
    }

    private int size;
    // 虚拟头节点
    private ListNode head;

    public MyLinkedList() {
        this.size = 0;
        this.head = new ListNode(0);
    }
    
    public int get(int index) {
        if (index < 0 || index >= size) {
            return -1;
        }

        ListNode current = head;
        for (int i = 0; i <= index; i++) {
            current = current.next;
        }

        return current.val;
    }
    
    public void addAtHead(int val) {
        ListNode originalHead = head.next;
        ListNode newListNode = new ListNode(val);
        head.next = newListNode;
        head.next.next = originalHead;
        size++;
    }
    
    public void addAtTail(int val) {
        ListNode current = head;
        while (current.next != null) {
            current = current.next;
        }

        current.next = new ListNode(val);
        size++;
    }
    
    public void addAtIndex(int index, int val) {
        if (index < 0 || index > size) {
            return;
        }

        ListNode current = head;
        for (int i = 0; i < index; i++) {
            current = current.next;
        }

        ListNode tmp = current.next;
        current.next = new ListNode(val);
        current.next.next = tmp;
        size++;
    }
    
    public void deleteAtIndex(int index) {
        if (index < 0 || index >= size) {
            return;
        }

        ListNode current = head;
        for (int i = 0; i < index; i++) {
            current = current.next;
        }
        current.next = current.next.next;
        size--;
    }
}

/**
 * Your MyLinkedList object will be instantiated and called as such:
 * MyLinkedList obj = new MyLinkedList();
 * int param_1 = obj.get(index);
 * obj.addAtHead(val);
 * obj.addAtTail(val);
 * obj.addAtIndex(index,val);
 * obj.deleteAtIndex(index);
 */
```

## 206\. Reverse Linked List

因为是单向链表，头疼的就是把当前节点的next改掉之后，后面的节点就访问不到了。解题思路是双指针，read作为读指针从链表的head开始往后遍历，prev作为跟屁虫，从read指针的前一个位置出发，跟着read走（也相当于从“新”链表的尾部开始，指示read指针指向的节点该把next改到哪）。修改期间，因为节点的next指向的位置会丢失，所以得用个临时对象先保存好，就跟交换两个变量的值类似。

```java
class Solution {
    public ListNode reverseList(ListNode head) {
        // 一开始head的前一个节点啥也没有，所以从null开始
        // 从结果来看，这也是“新”链表最后一个节点之后的东西，当然也是null
        ListNode prev = null;
        ListNode read = head;

        while (read != null) {
            // 比如链表[1, 2, 3, 4, 5]
            // 遍历到1的时候，先把2的引用存起来
            final ListNode tmp = read.next;
            // 然后把当前节点的next指向它的前一个节点
            // 也就是反转
            read.next = prev;
            // 反转完了之后，指向“新”链表尾部的指针指向当前节点
            prev = read;
            // 读指针指向原链表的下一个节点，也就是之前保存的临时变量
            read = tmp;
        }

        return prev;
    }
}
```

## 19\. Remove Nth Node From End of List

这道题有两个解法，一个方法是先全部遍历一次得到链表长度，再计算出要删除的节点的位置；另一个方法是双指针。暴力解没啥难度，只记一下双指针的思路。

双指针解法的关键点是，在开始的时候要让`fast`指针先走n步，来保证`fast`和`slow`指针之间固定保持n个节点的距离，这样，在`fast`抵达尾部后，`slow`正好距尾部有n个节点，也就是位于倒数第n的位置。想到这，其实让`fast`走n+1步更合适，因为此时，`slow`就正好位于要被删除节点的前一个位置。

```java
class Solution {
    public ListNode removeNthFromEnd(ListNode head, int n) {
        // 定义一个虚拟头节点，省得处理被删掉的是头节点之类的麻烦事
        ListNode dummyHead = new ListNode();
        dummyHead.next = head;

        ListNode slow = dummyHead;
        ListNode fast = dummyHead;

        // 让fast先走n+1步
        for (int i = 0; i <= n; i++) {
            fast = fast.next;
        }

        // 然后一起往后走直到fast到了末尾
        while (fast != null) {
            slow = slow.next;
            fast = fast.next;
        }

        // 因为slow与fast永远隔n+1个位置
        // 那么fast走到头，slow就位于从后往前数第n+1个位置
        // 也就是倒数第n+1个节点
        slow.next = slow.next.next;

        return dummyHead.next;
    }
}
```

## (160. Intersection of Two Linked Lists)\[https://leetcode.com/problems/intersection-of-two-linked-lists/\]

找两个链表相交的位置，也就是找两个链表中第一个相同的节点，而且比较的不是值相同而是引用相同，那Java仔第一反应就是`把Set给我端上来！`  
思路也很简单，先遍历第一条链表，把节点全放进一个Set里，然后遍历第二条，逐个检查Set里是不是包含这个对象。

```java
public class Solution {
    public ListNode getIntersectionNode(ListNode headA, ListNode headB) {
        final HashSet<ListNode> set = new HashSet<>();

        ListNode head = headA;
        while (head != null) {
            set.add(head);
            head = head.next;
        }

        head = headB;
        while (head != null) {
            if (set.contains(head)) {
                return head;
            }
            head = head.next;
        }

        return null;
    }
}
```

但写完了感觉这不像是LeetCode的调性，既然有两个链表，那得有俩指针嘁哩喀喳一顿算，再巧妙地找到两者的焦点。于是看了眼攻略，又有了这个思路：

> 先算出来两条链表的长度，得到两条链表长度之差，然后把较长的那条链表开头那几个“多余的”节点砍掉，让两条链表一样长，接下来从同一起跑线开始走，逐个比较二者的对象是否相同。

嗯，这才是符合我对LeetCode刻板印象的解法。

```java
public class Solution {
    public ListNode getIntersectionNode(ListNode headA, ListNode headB) {
        int sizeA = 0;
        int sizeB = 0;

        // 算出两个链表的长度
        ListNode head = headA;
        while (head != null) {
            sizeA++;
            head = head.next;
        }

        head = headB;
        while (head != null) {
            sizeB++;
            head = head.next;
        }

        // 计算长度之差
        int diff = sizeA - sizeB;

        // lenA - lenB小于0，说明A比B短
        // 那么就把B链表多出来的节点拿掉
        while (diff < 0) {
            headB = headB.next;
            diff++;
        }
        // 反之亦然
        // 当然这两个while实际只有一个会执行
        // 俩都执行只有一种可能：你的内存被宇宙射线轰出比特翻转还正好翻转在这了
        // 别做题了买彩票去吧
        while (diff > 0) {
            headA = headA.next;
            diff--;
        }

        // 现在两条链表一样长，从谁开始无所谓
        // 只要同步一起往下走，保证能一起走到终点
        // （不，这不是什么一起白了头的爱情小故事
        while (headA != null) {
            // 如果俩对象一样，那就找到了交叉点
            if (headA == headB) {
                return headA;
            }

            headA = headA.next;
            headB = headB.next;
        }

        return null;
    }
}
```

## 142\. Linked List Cycle II

找环形链表中入环的第一个节点，类似找两个链表相交的位置一样，就是看哪个节点出现了两次。自然，用Set做这件事就很合适，而且代码一眼就看懂。

```java
public class Solution {
    public ListNode detectCycle(ListNode head) {
        final Set<ListNode> set = new HashSet<>();

        ListNode current = head;
        while (current != null) {
            if (set.contains(current)) {
                return current;
            }

            set.add(current);
            current = current.next;
        }

        return null;
    }
}
```

但还是那句话，这解法不LeetCode。看完攻略我才知道，有个叫`Floyd判圈算法`专门解决这个问题。在这个算法中会用到快慢两个指针，慢指针一次走一步，快指针一次走两步。

此外，在这个算法中还需要用到下面这些长度：

-   从头节点到环入口的距离记为`a`
-   从入口到快慢指针相遇点的距离记为`b`
-   从相遇点继续往前走到环入口的距离记为`c`
-   环的长度`L`就是`b + c`
-   因为快指针先入环，需要绕环`n`圈才会与慢指针相遇

那么相遇时，慢指针走了`a + b`步，快指针走了`a + b + nL`。又因为快指针速度是慢指针的2倍，即`2*(a + b)`，所以有下面等式：

```plaintext
2 * (a + b) = a + b + nL
      a + b = nL
          a = nL - b
          a = (n - 1)L + (L - b)
          a = (n - 1)L + (b + c - b)
          a = (n - 1)L + c
```

上面等式中，`(n - 1)L`是在环中白绕的圈，不会改变相遇点的位置。也就是说，从相遇点出发再走`c`步，就能到达环入口。结合`a`是从头节点到环入口的距离，就可知道，在相遇后让两个指针分别从链表头和相遇点开始每次各走一步，两个指针必将在环入口相遇。

那么根据这个思路，就有了下面的代码：

```java
public class Solution {
    public ListNode detectCycle(ListNode head) {
        ListNode slow = head;
        ListNode fast = head;

        // 快慢指针找相遇点
        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;

            if (slow == fast) {
                ListNode ptr1 = head;
                ListNode ptr2 = fast;

                while (ptr1 != ptr2) {
                    ptr1 = ptr1.next;
                    ptr2 = ptr2.next;
                }

                return ptr1;
            }
        }

        return null;
    }
}
```

## 349\. Intersection of Two Arrays

找两个数组的交集，都不用想，肯定是先把一个数组放进Set里，然后拿另一个数组过来找有没有，找到就放进另一个Set里，这第二个Set就是两个数组的交集。

```java
class Solution {
    public int[] intersection(int[] nums1, int[] nums2) {
        final Set<Integer> set = new HashSet<>();
        final Set<Integer> resultSet = new HashSet<>();

        // 反正求交集，从谁开始无所谓
        for (int i : nums1) {
            set.add(i);
        }

        for (int i : nums2) {
            if (set.contains(i)) {
                resultSet.add(i);
            }
        }

        final int[] resultArr = new int[resultSet.size()];
        int index = 0;
        for (Integer i : resultSet) {
            resultArr[index] = i;
            index++;
        }

        return resultArr;
    }
}
```
