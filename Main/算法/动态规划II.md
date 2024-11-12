# 动态规划II

## 线性DP

### 最长公共子序列（LCS）

#### [最长公共子序列](https://leetcode.cn/problems/longest-common-subsequence/)

给定两个字符串 `text1` 和 `text2`，返回这两个字符串的最长公共子序列的长度。

[判断子序列](https://leetcode.cn/problems/is-subsequence/) 可用相同逻辑。

`dp[i][j]`：text1[0 : i] 和 text2[0 : j] 的最长公共子序列的长度。

```java
public int longestCommonSubsequence(String text1, String text2) {
    int[][] dp = new int[text1.length() + 1][text2.length() + 1];

    for (int i = 1; i <= text1.length(); i++) {
        for (int j = 1; j <= text2.length(); j++) {
            if (text1.charAt(i - 1) == text2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    return dp[text1.length()][text2.length()];
}
```

空间优化：

```java
public int longestCommonSubsequence(String text1, String text2) {
    int[] dp = new int[text2.length() + 1];

    for (int i = 1; i <= text1.length(); i++) {
        int pre = 0;
        for (int j = 1; j <= text2.length(); j++) {
            int temp = dp[j];
            if (text1.charAt(i - 1) == text2.charAt(j - 1)) {
                dp[j] = pre + 1;
            } else {
                dp[j] = Math.max(dp[j], dp[j - 1]);
            }
            pre = temp;
        }
    }
    return dp[text2.length()];
}
```

#### [编辑距离](https://leetcode.cn/problems/edit-distance/)

给你两个单词 `word1` 和 `word2`， 请返回将 `word1` 转换成 `word2` 所使用的最少操作数 。

```java
public int minDistance(String word1, String word2) {
    int[][] dp = new int[word1.length() + 1][word2.length() + 1];
    for (int i = 0; i <= word1.length(); i++) {
        dp[i][0] = i;
    }
    for (int i = 0; i <= word2.length(); i++) {
        dp[0][i] = i;
    }

    for (int i = 1; i <= word1.length(); i++) {
        for (int j = 1; j <= word2.length(); j++) {
            if (word1.charAt(i - 1) == word2.charAt(j - 1)) {
                dp[i][j] = dp[i - 1][j - 1];
            } else {
                dp[i][j] = Math.min(Math.min(dp[i - 1][j], dp[i][j - 1]), dp[i - 1][j - 1]) + 1;
            }
        }
    }
    return dp[word1.length()][word2.length()];
}
```



### 最长递增子序列（LIS）

#### [最长递增子序列](https://leetcode.cn/problems/longest-increasing-subsequence/)

给你一个整数数组 `nums` ，找到其中最长严格递增子序列的长度。

> **子序列** 是由数组派生而来的序列，删除（或不删除）数组中的元素而不改变其余元素的顺序。

##### 动态规划

`dp[i]`：nums 以 nums[i] 结尾的最长子序列长度

```java
public int lengthOfLIS(int[] nums) {
    int[] dp = new int[nums.length];
    int res = 0;

    for (int i = 0; i < nums.length; i++) {
        int max = 1;
        for (int j = 0; j < i; j++) { // 记录i之前最长递增长度
            if (nums[j] < nums[i]) {
                max = Math.max(max, dp[j] + 1);
            }
        }
        dp[i] = max;
        res = Math.max(res, dp[i]);
    }
    return res;
}
```

##### 二分查找

```java
public int lengthOfLIS(int[] nums) {
    List<Integer> res = new ArrayList<>();
    for (int num : nums) {
        int index = binarySearch(res, num);
        if (index < res.size()) {
            res.set(index, num); // *
        } else {
            res.add(num);
        }
    }
    return res.size();
}

private int binarySearch(List<Integer> res, int target) {
    int left = 0;
    int right = res.size() - 1;
    while (left <= right) {
        int mid = (left + right) >>> 1;
        if (res.get(mid) < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    return left;
}
```

遍历 nums 找到当前 res 尾部更小的数，就把它换进来，要是后续不能让这些稍小（相较于 res 的尾部）的变成更长的子序列，那就超不过原先的，也没有任何损失。



#### [最长递增子序列的个数](https://leetcode.cn/problems/number-of-longest-increasing-subsequence/)

给定一个未排序的整数数组 `nums` ， 返回最长递增子序列的个数 。

`dp[i]`：nums 以 nums[i] 结尾的最长子序列长度。

`count[i]`：以 nums[i] 结尾的最长上升子序列的个数。

```java
public int findNumberOfLIS(int[] nums) {
    int[] dp = new int[nums.length];
    int[] count = new int[nums.length];
    Arrays.fill(dp, 1);
    Arrays.fill(count, 1);
    int max = 0;
    for (int i = 0; i < nums.length; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[j] < nums[i]) {
                if (dp[i] < dp[j] + 1) {
                    dp[i] = dp[j] + 1;
                    count[i] = count[j];
                } else if (dp[i] == dp[j] + 1) {
                    count[i] += count[j];
                }
            }
        }
        max = Math.max(max, dp[i]);
    }

    int res = 0;
    for (int i = 0; i < nums.length; i++) {
        if (dp[i] == max) {
            res += count[i];
        }
    }
    return res;
}
```



#### [俄罗斯套娃信封问题](https://leetcode.cn/problems/russian-doll-envelopes/)

当另一个信封的宽度和高度都比这个信封大的时候，这个信封就可以放进另一个信封里，如同俄罗斯套娃一样。

信封嵌套问题就需要先按特定的规则排序，之后就转换为一个最长递增子序列问题。

##### 动态规划（超时）

```java
public int maxEnvelopes(int[][] envelopes) {
    Arrays.sort(envelopes, new Comparator<int[]>() {
        @Override
        public int compare(int[] a, int[] b) {
            return a[0] == b[0] ? b[1] - a[1] : a[0] - b[0];
        }
    });
    int[] dp = new int[envelopes.length];
    int res = 0;
    for (int i = 0; i < envelopes.length; i++) {
        int max = 1;
        for (int j = 0; j < i; j++) {
            if (envelopes[j][0] < envelopes[i][0] && envelopes[j][1] < envelopes[i][1]) {
                max = Math.max(max, dp[j] + 1);
            }
        }
        dp[i] = max;
        res = Math.max(res, dp[i]);
    }
    return res;
}
```

先对宽度进行升序排序，如果遇到 宽度 相同的情况，则按照高度降序排序。

##### 二分查找

```java
public int maxEnvelopes(int[][] envelopes) {
    Arrays.sort(envelopes, new Comparator<int[]>() {
        @Override
        public int compare(int[] a, int[] b) {
            return a[0] == b[0] ? b[1] - a[1] : a[0] - b[0];
        }
    });
    List<Integer> res = new ArrayList<>();
    for (int i = 0; i < envelopes.length; i++) {
        int num = envelopes[i][1];
        int index = binarySearch(res, num);
        if (index == res.size()) {
            res.add(num);
        } else {
            res.set(index, num);
        }
    }
    return res.size();
}

private int binarySearch(List<Integer> res, int target) {
    int left = 0;
    int right = res.size() - 1;
    while (left <= right) {
        int mid = (left + right) >>> 1;
        if (res.get(mid) < target) {
            left = mid + 1;
        } else if (res.get(mid) > target) {
            right = mid - 1;
        } else {
            return mid;
        }
    }
    return left;
}
```



## 区间 DP

### [最长回文子序列](https://leetcode.cn/problems/longest-palindromic-subsequence/)

给你一个字符串 `s` ，找出其中最长的回文子序列，并返回该序列的长度。

> 子序列定义为：不改变剩余字符顺序的情况下，删除某些字符或者不删除任何字符形成的一个序列。

`dp[i][j]`：考虑区间 [i, j] 的最长回文子序列长度为多少。

```java
public int longestPalindromeSubseq(String s) {
    int[][] dp = new int[s.length()][s.length()];
    for (int i = s.length() - 1; i >= 0; i--) { // 倒序
        dp[i][i] = 1;
        for (int j = i + 1; j < s.length(); j++) {
            if (s.charAt(i) == s.charAt(j)) {
                dp[i][j] = dp[i + 1][j - 1] + 2;
            } else {
                dp[i][j] = Math.max(dp[i + 1][j], dp[i][j - 1]);
            }
        }
    }
    return dp[0][s.length() - 1];
}
```

递推公式`dp[i][j] = dp[i + 1][j - 1] + 2` 和 `dp[i][j] = max(dp[i + 1][j], dp[i][j - 1])` 可以看出，`dp[i][j]`是依赖于`dp[i + 1][j - 1] 和 dp[i + 1][j]`，也就是从矩阵的角度来说，`dp[i][j]` 下一行的数据。 所以遍历i的时候一定要从下到上遍历，这样才能保证，下一行的数据是经过计算的。



### [回文子串](https://leetcode.cn/problems/palindromic-substrings/)

给你一个字符串 `s` ，请你统计并返回这个字符串中 **回文子串** 的数目。

> **回文字符串** 是正着读和倒过来读一样的字符串。
>
> **子字符串** 是字符串中的由连续字符组成的一个序列。

#### 动态规划

`dp[i][j]`：子串 s[i : j] 是否为回文子串，这里子串 s[i : j] 定义为左闭右闭区间，即可以取到 s[i] 和 s[j]。

方式一：

```java
public int countSubstrings(String s) {
    int res = 0;
    boolean[][] dp = new boolean[s.length()][s.length()];
    for (int i = s.length() - 1; i >= 0; i--) { // 倒序
        for (int j = i; j < s.length(); j++) {
            if (s.charAt(i) == s.charAt(j)) {
                if (j - i < 3 || dp[i + 1][j - 1]) {
                    count++;
                    dp[i][j] = true;
                }
            }
        }
    }
    return res;
}
```

方式二：

```java
public int countSubstrings(String s) {
    int res = 0;
    boolean[][] dp = new boolean[s.length()][s.length()];
    for (int j = 0; j < s.length(); j++) { // 先遍历j
        for (int i = 0; i <= j; i++) {
            if (s.charAt(i) == s.charAt(j)) {
                if (j - i < 3 || dp[i + 1][j - 1]) {
                    count++;
                    dp[i][j] = true;
                }
            }
        }
    }
    return res;
}
```

#### 中心扩散法

分别考虑子串的长度为奇数和偶数的情况。

```java
public int countSubstrings(String s) {
    int count = 0;
    for (int i = 0; i < s.length(); i++) {
        count += getCount(s, i, i);
        count += getCount(s, i, i + 1);
    }
    return count;
}

public int getCount(String s, int left, int right) {
    int count = 0;
    while (left >= 0 && right <= s.length() - 1 && s.charAt(left) == s.charAt(right)) {
        left--;
        right++;
        count++;
    }
    return count;
}
```



### [最长回文子串](https://leetcode.cn/problems/longest-palindromic-substring/)

给你一个字符串 `s`，找到 `s` 中最长的 回文子串。

#### 动态规划

逻辑同 [回文子串](https://leetcode.cn/problems/palindromic-substrings/)。

`dp[i][j]`：子串 s[i : j] 是否为回文子串，这里子串 s[i : j] 定义为左闭右闭区间，即可以取到 s[i] 和 s[j]。

```java
public String longestPalindrome(String s) {
    int maxLen = 1;
    int begin = 0;
    boolean[][] dp = new boolean[s.length()][s.length()];
    for (int i = s.length() - 1; i >= 0; i--) { // 倒序
        for (int j = i; j < s.length(); j++) {
            if (s.charAt(i) == s.charAt(j)) {
                if (j - i < 3 || dp[i + 1][j - 1]) {
                    dp[i][j] = true;
                }
            }

            if (dp[i][j] && j - i + 1 > maxLen) { // dp[i][j]为true时记录回文长度和起始位置
                maxLen = j - i + 1;
                begin = i;
            }
        }
    }
    return s.substring(begin, begin + maxLen);
}
```

#### 中心扩散法

方式一：

```java
public String longestPalindrome(String s) {
    char[] arr = s.toCharArray();
    int begin = 0;
    int maxLen = 0;
    for (int i = 0; i < arr.length; i++) {
        int left = i - 1;
        int right = i + 1;
        int len = 1;

        while (left >= 0 && arr[left] == arr[i]) {
            left--;
            len++;
        }
        while (right <= arr.length - 1 && arr[right] == arr[i]) {
            right++;
            len++;
        }
        while (left >= 0 && right <= arr.length - 1 && arr[left] == arr[right]) {
            left--;
            right++;
            len += 2;
        }
        if (len > maxLen) {
            begin = left + 1;
            maxLen = len;
        }
    }
    return s.substring(begin, begin + maxLen);
}
```

方式二：

逻辑同 [回文子串](https://leetcode.cn/problems/palindromic-substrings/)。

```java
public String longestPalindrome(String s) {
    int maxLen = 0;
    int begin = 0;
    for (int i = 0; i < s.length(); i++) {
        Result oddResult = getLongest(s, i, i);
        Result evenResult = getLongest(s, i, i + 1);

        Result result = oddResult.len > evenResult.len ? oddResult : evenResult;
        if (maxLen < result.len) {
            maxLen= result.len;
            begin = result.begin;
        }
    }
    return s.substring(begin, begin + maxLen);
}

public Result getLongest(String s, int left, int right) {
    while (left >= 0 && right <= s.length() - 1 && s.charAt(left) == s.charAt(right)) {
        left--;
        right++;
    }
    return new Result(left + 1, right - left - 1);
}

class Result {
    int begin;
    int len;

    public Result(int begin, int len) {
        this.begin = begin;
        this.len = len;
    }
}
```



### [分割回文串 II](https://leetcode.cn/problems/palindrome-partitioning-ii/)

给你一个字符串 `s`，将 `s` 分割成一些子串，使每个子串都是回文串。返回符合要求的最少分割次数 。

通过 [回文子串](https://leetcode.cn/problems/palindromic-substrings/) 获取dp数组用于判断区间是否为回文。

`dp[i]`：字符串的前缀 s[0..i]的最少分割次数。

```java
public int minCut(String s) {
    boolean[][] isPalindrome = getPalindromeCheckArray(s);

    int[] dp = new int[s.length()];
    for (int i = 0; i < s.length(); i++) {
        dp[i] = i;
    }
    for (int index = 0; index < s.length(); index++) {
        if (isPalindrome[0][index]) {
            dp[index] = 0;
            continue;
        }

        for (int i = 0; i < index; i++) {
            if (isPalindrome[i + 1][index]) {
                dp[index] = Math.min(dp[index], dp[i] + 1);
            }
        }
    }
    return dp[s.length() - 1];
}

public boolean[][] getPalindromeCheckArray(String s) {
    boolean[][] dp = new boolean[s.length()][s.length()];
    for (int i = s.length() - 1; i >= 0; i--) { // 倒序
        for (int j = i; j < s.length(); j++) {
            if (s.charAt(i) == s.charAt(j)) {
                if (j - i <= 2 || dp[i + 1][j - 1]) {
                    dp[i][j] = true;
                }
            }
        }
    }
    return dp;
}
```



## 划分型 DP

### [单词拆分](https://leetcode.cn/problems/word-break/)

给你一个字符串 `s` 和一个字符串列表 `wordDict` 作为字典。如果可以利用字典中出现的一个或多个单词拼接出 `s` 则返回 true。不要求字典中出现的单词全部都使用，并且字典中的单词可以重复使用。

**完全背包问题**，求的是排列数，外循环遍历 `target`，内循环遍历 `nums`（内循环正序）。

`dp[i]`：字符串 s 前 i 个字符组成的字符串 s[0 : i−1] 是否能划分。

```java
public boolean wordBreak(String s, List<String> wordDict) {
    boolean[] dp = new boolean[s.length() + 1];
    dp[0] = true;
    for (int i = 1; i <= s.length(); i++) {
        for (String word : wordDict) {
            if (i - word.length() >= 0 && s.substring(i - word.length(), i).equals(word)) {
                dp[i] = dp[i] || dp[i - word.length()];
            }
        }
    }
    return dp[s.length()];
}
```



## 数位DP

- `isLimit` 表示当前是否受到了 n 的约束。
  - 若为真，则第 i 位填入的数字至多为 s[i]。如果在受到约束的情况下填了 s[i]，那么后续填入的数字仍会受到 n 的约束。
  - 若为真，填入的数字可以从 0 开始到 9。
- `hasNum` 表示 i 前面的数位是否填了数字（前导零）。
  - 若为假，则当前位可以不填数字跳过，如果要填入的数字至少为 1。
  - 若为真，则要填入的数字可以从 0 开始。



### [至少有 1 位重复的数字](https://leetcode.cn/problems/numbers-with-repeated-digits/)

换成求无重复数字的个数，答案等于 *n* 减去无重复数字的个数。

> 位运算与集合论
>
> 集合可以用二进制表示，二进制从低到高第 i 位为 1 表示 i 在集合中，为 0 表示 i 不在集合中。
>
> 例如集合 {0,2,3} 对应的二进制数为 1101。
>
> 设集合对应的二进制数为 x。
>
> - 判断元素 d 是否在集合中：`x >> d & 1` 可以取出 x 的第 d 个比特位，如果是 1 就说明 d 在集合中。
> - 把元素 d 添加到集合中：将 x 更新为 `x | (1 << d)`。

**算法模板**。

可以只记录不受到 isLimit 或 isNum 约束时的状态 (i, mask)。

- 比如 n=234，第一位填 2，第二位填 3，后面无论怎么递归，都不会再次递归到第一位填 2，第二位填 3 的情况，所以不需要记录。

- 比如第一位不填，第二位也不填，后面无论怎么递归也不会再次递归到这种情况，所以也不需要记录。

```java
private char[] arr;
private int[][] memory;

public int numDupDigitsAtMostN(int n) {
    arr = Integer.toString(n).toCharArray();
    memory = new int[arr.length][1 << 10];
    for (int i = 0; i < arr.length; i++) {
        Arrays.fill(memory[i], -1);
    }

    return n - dfs(0, 0, true, false);
}

int dfs(int index, int mask, boolean isLimit, boolean hasNum) {
    if (index == arr.length) {
        return hasNum ? 1 : 0;
    }
    if (!isLimit && hasNum) {
        if (memory[index][mask] != -1) {
            return memory[index][mask];
        }
    }

    int res = 0;
    if (!hasNum) {
        res = dfs(index + 1, mask, false, false); // 可以跳过当前数位
    }

    int upperBound = isLimit ? arr[index] - '0' : 9;
    for (int i = hasNum ? 0 : 1; i <= upperBound; i++) {
        if ((mask >> i & 1) == 0) {
            res += dfs(index + 1, mask | (1 << i), isLimit && (i == upperBound), true);
        }
    }
    if (!isLimit && hasNum) {
        memory[index][mask] = res;
    }
    return res;
}
```



### [数字 1 的个数](https://leetcode.cn/problems/number-of-digit-one/)

本题前导零对答案无影响，`hasNum` 可以省略。

```java
private char[] arr;
private int[][] memory;

public int countDigitOne(int n) {
    arr = Integer.toString(n).toCharArray();
    memory = new int[arr.length][arr.length];
    for (int[] row : memory) {
        Arrays.fill(row, -1);
    }
    return dfs(0, 0, true);
}

private int dfs(int index, int count, boolean isLimit) {
    if (index == arr.length) {
        return count;
    }
    if (!isLimit && memory[index][count] >= 0) {
        return memory[index][count];
    }

    int res = 0;
    int upperBound = isLimit ? arr[index] - '0' : 9;
    for (int i = 0; i <= upperBound; i++) {
        res += dfs(index + 1, count + (i == 1 ? 1 : 0), isLimit && (i == upperBound));
    }
    if (!isLimit) {
        memory[index][count] = res;
    }
    return res;
}
```



## 网格DP

网格问题可以使用记忆化 DFS 求解。

### [不同路径](https://leetcode.cn/problems/unique-paths/)

一个机器人位于一个 m x n 网格的左上角 。机器人每次只能向下或者向右移动一步。机器人试图达到网格的右下角。问总共有多少条不同的路径？

#### 动态规划

```java
public int uniquePaths(int m, int n) {
    int[][] dp = new int[m][n];
    for (int i = 0; i < m; i++) {
        dp[i][0] = 1;
    }
    for (int i = 0; i < n; i++) {
        dp[0][i] = 1;
    }

    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
        }
    }
    return dp[m - 1][n - 1];
}
```

#### 记忆化DFS

```java
public int uniquePaths(int m, int n) {
    int[][] memory = new int[m + 1][n +1];
    for (int i = 0; i < memory.length; i++) {
        for (int j = 0; j < memory[0].length; j++) {
            memory[i][j] = Integer.MAX_VALUE;
        }
    }
    return dfs(1, 1, m, n, memory);
}

public int dfs(int row, int col, int rowLen, int colLen, int[][] memory) {
    if (row > rowLen || col > colLen) {
        return 0;
    }
    if (row == rowLen && col == colLen) {
        return 1;
    }

    if (memory[row][col] != Integer.MAX_VALUE) {
        return memory[row][col];
    }
    int res = dfs(row + 1, col, rowLen, colLen, memory) + dfs(row, col + 1, rowLen, colLen, memory);
    memory[row][col] = res;
    return res ;
}
```

### [不同路径 II](https://leetcode.cn/problems/unique-paths-ii/)

考虑网格中有障碍物。网格中的障碍物和空位置分别用 1 和 0 来表示。

#### 动态规划

```java
public int uniquePathsWithObstacles(int[][] obstacleGrid) {
    int rowLen = obstacleGrid.length;
    int colLen = obstacleGrid[0].length;
    int[][] dp = new int[rowLen][colLen];

    if (obstacleGrid[0][0] == 1) {
        return 0;
    }
    for (int i = 0; i < rowLen; i++) {
        if (obstacleGrid[i][0] == 1) {
            break;
        }
        dp[i][0] = 1;
    }
    for (int i = 0; i < colLen; i++) {
        if (obstacleGrid[0][i] == 1) {
            break;
        }
        dp[0][i] = 1;
    }

    for (int i = 1; i < rowLen; i++) {
        for (int j = 1; j < colLen; j++) {
            if (obstacleGrid[i][j] == 1) {
                dp[i][j] = 0;
            } else {
                dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
            }
        }
    }
    return dp[rowLen - 1][colLen - 1];
}
```

#### 记忆化DFS

```java
public int uniquePathsWithObstacles(int[][] obstacleGrid) {
    int[][] memory = new int[obstacleGrid.length][obstacleGrid[0].length];
    for (int i = 0; i < obstacleGrid.length; i++) {
        for (int j = 0; j < obstacleGrid[0].length; j++) {
            memory[i][j] = Integer.MAX_VALUE;
        }
    }
    return dfs(obstacleGrid, 0, 0, memory);
}

public int dfs(int[][] obstacleGrid, int row, int col, int[][] memory) {
    if (row > obstacleGrid.length - 1 || col > obstacleGrid[0].length - 1) {
        return 0;
    }
    if (obstacleGrid[row][col] == 1) {
        return 0;
    }
    if (memory[row][col] != Integer.MAX_VALUE) {
        return memory[row][col];
    }

    if (row == obstacleGrid.length - 1 && col == obstacleGrid[0].length - 1) {
        return 1;
    }

    int res = dfs(obstacleGrid, row, col + 1, memory) 
        + dfs(obstacleGrid, row + 1, col, memory);

    memory[row][col] = res;
    return res;
}
```
















