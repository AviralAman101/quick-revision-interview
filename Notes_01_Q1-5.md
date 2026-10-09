# Notes 1 of 6: Questions 1 to 5

Arrays, hashing, sliding window and intervals. Java 8 compatible.

> Reconstructed from the topic list you gave me (the original session statements and your submitted code were not retained). The statements and solutions are standard, clean reference versions, not your exact submissions.

---

## Q1. Minimum Adjacent Swaps to Group 1s

**Part A.** `nums` is a binary array. In one move you swap two adjacent elements. Return the minimum moves so that `k` ones are consecutive. If `k` is the total number of ones, this groups all the ones together.

```
nums=[1,0,0,1,0,1], k=2          -> 1
nums=[1,0,0,0,0,0,1,1], k=3      -> 5
nums=[1,1,0,1], k=2              -> 0
```

**Part B.** Return the minimum adjacent swaps to group **all** the 1s together.

```
[1,0,1,0,1]     -> 1
[0,1,0,0,1,0,1] -> 2
[1,1,1]         -> 0
```

**Idea.** Take the positions `p[]` of the ones. If the final block starts at some place, the `i`-th one lands at `start + i`. Define `q[i] = p[i] - i`. Then the cost is the sum of `|q[i] - target|` over the window, which is minimised when `target` is the median of the window. Slide a window of size `k` and use prefix sums of `q` to get the cost in O(1) per window.

```java
public static int minMoves(int[] nums, int k) {
    List<Integer> pos = new ArrayList<>();
    for (int i = 0; i < nums.length; i++) if (nums[i] == 1) pos.add(i);
    int m = pos.size();
    long[] q = new long[m];
    long[] pre = new long[m + 1];
    for (int i = 0; i < m; i++) {
        q[i] = pos.get(i) - i;
        pre[i + 1] = pre[i] + q[i];
    }
    long best = Long.MAX_VALUE;
    for (int l = 0; l + k <= m; l++) {
        int mid = l + k / 2;
        long med = q[mid];
        long left = med * (mid - l) - (pre[mid] - pre[l]);
        long right = (pre[l + k] - pre[mid + 1]) - med * (l + k - 1 - mid);
        best = Math.min(best, left + right);
    }
    return (int) best;
}

public static int minSwapsAllOnes(int[] nums) {
    int ones = 0;
    for (int x : nums) ones += x;
    if (ones <= 1) return 0;
    return minMoves(nums, ones);
}
```

**Complexity.** O(n) time, O(m) space, where `m` is the number of ones.

---

## Q2. Two Sum and Variants

**Part A.** Return the indices of the two numbers that add up to `target`. Exactly one solution exists and you may not use the same element twice.

```
[2,7,11,15], target=9 -> [0,1]
[3,2,4],     target=6 -> [1,2]
[3,3],       target=6 -> [0,1]
```

**Part B (Two Sum II).** The array is sorted ascending. Return the 1-indexed positions of the two numbers, using O(1) extra space.

```
[2,7,11,15], target=9 -> [1,2]
[2,3,4],     target=6 -> [1,3]
[-1,0],      target=-1 -> [1,2]
```

**Idea.** Hash map from value to index: for each `x`, check whether `target - x` was already seen. Sorted input allows two pointers instead.

```java
public static int[] twoSum(int[] nums, int target) {
    Map<Integer, Integer> seen = new HashMap<>();
    for (int i = 0; i < nums.length; i++) {
        Integer j = seen.get(target - nums[i]);
        if (j != null) return new int[]{j, i};
        seen.put(nums[i], i);
    }
    return new int[0];
}

public static int[] twoSumSorted(int[] numbers, int target) {
    int l = 0, r = numbers.length - 1;
    while (l < r) {
        int s = numbers[l] + numbers[r];
        if (s == target) return new int[]{l + 1, r + 1};
        if (s < target) l++; else r--;
    }
    return new int[0];
}
```

**Complexity.** Part A: O(n) time and space. Part B: O(n) time, O(1) space.

---

## Q3. Maximum Subarray (Kadane) and Circular Version

**Part A.** Return the largest sum of a non-empty contiguous subarray.

```
[-2,1,-3,4,-1,2,1,-5,4] -> 6   ([4,-1,2,1])
[1]                     -> 1
[-3,-1,-2]              -> -1
```

**Part B.** The array is circular, so a subarray may wrap around the end. Return the largest possible sum.

```
[1,-2,3,-2]  -> 3
[5,-3,5]     -> 10
[-3,-2,-3]   -> -2
```

**Idea.** Kadane: `cur = max(x, cur + x)`. For the circular case, the best wrapping subarray equals `total - (minimum subarray)`. The answer is the larger of the normal best and that value, except when every element is negative (then the wrapping value would be 0, an empty subarray), so return the normal best.

```java
public static int maxSubArray(int[] nums) {
    int cur = nums[0], best = nums[0];
    for (int i = 1; i < nums.length; i++) {
        cur = Math.max(nums[i], cur + nums[i]);
        best = Math.max(best, cur);
    }
    return best;
}

public static int maxSubarraySumCircular(int[] nums) {
    int total = 0;
    int curMax = 0, maxSum = nums[0];
    int curMin = 0, minSum = nums[0];
    for (int x : nums) {
        total += x;
        curMax = Math.max(x, curMax + x);
        maxSum = Math.max(maxSum, curMax);
        curMin = Math.min(x, curMin + x);
        minSum = Math.min(minSum, curMin);
    }
    if (maxSum < 0) return maxSum;           // all negative
    return Math.max(maxSum, total - minSum);
}
```

**Complexity.** O(n) time, O(1) space.

---

## Q4. Longest Substring Without Repeating Characters

**Part A.** Return the length of the longest substring with no repeated characters.

```
"abcabcbb" -> 3
"bbbbb"    -> 1
"pwwkew"   -> 3
""         -> 0
```

**Part B.** Return the length of the longest substring with at most `k` distinct characters.

```
"eceba", k=2   -> 3   ("ece")
"aa",    k=1   -> 2
"abaccc", k=2  -> 4   ("accc")
```

**Idea.** Sliding window. Part A keeps a map of each character's last seen index: when a repeat is inside the window, jump `left` to one past it. Part B keeps counts and shrinks the window while it has more than `k` distinct characters.

```java
public static int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> last = new HashMap<>();
    int left = 0, best = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        Integer prev = last.get(c);
        if (prev != null && prev >= left) left = prev + 1;
        last.put(c, right);
        best = Math.max(best, right - left + 1);
    }
    return best;
}

public static int lengthOfLongestSubstringKDistinct(String s, int k) {
    if (k == 0) return 0;
    Map<Character, Integer> count = new HashMap<>();
    int left = 0, best = 0;
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        count.merge(c, 1, Integer::sum);
        while (count.size() > k) {
            char l = s.charAt(left++);
            if (count.merge(l, -1, Integer::sum) == 0) count.remove(l);
        }
        best = Math.max(best, right - left + 1);
    }
    return best;
}
```

**Complexity.** O(n) time, O(alphabet) space.

---

## Q5. Merge Intervals (A) and Insert Interval (B)

**Part A.** Merge all overlapping intervals.

```
[[1,3],[2,6],[8,10],[15,18]] -> [[1,6],[8,10],[15,18]]
[[1,4],[4,5]]                -> [[1,5]]
[[1,4],[0,4]]                -> [[0,4]]
```

**Part B.** `intervals` is sorted and non-overlapping. Insert `newInterval` and merge if needed.

```
intervals=[[1,3],[6,9]], new=[2,5]                       -> [[1,5],[6,9]]
intervals=[[1,2],[3,5],[6,7],[8,10],[12,16]], new=[4,8]  -> [[1,2],[3,10],[12,16]]
intervals=[], new=[5,7]                                  -> [[5,7]]
```

**Idea.** Sort by start and sweep: if the next interval starts at or before the current end, extend the end, otherwise start a new interval. For insertion, copy intervals ending before the new one, merge the overlapping ones into it, then copy the rest.

```java
public static int[][] merge(int[][] intervals) {
    if (intervals.length == 0) return new int[0][];
    Arrays.sort(intervals, (a, b) -> Integer.compare(a[0], b[0]));
    List<int[]> res = new ArrayList<>();
    int[] cur = intervals[0];
    res.add(cur);
    for (int i = 1; i < intervals.length; i++) {
        if (intervals[i][0] <= cur[1]) {
            cur[1] = Math.max(cur[1], intervals[i][1]);
        } else {
            cur = intervals[i];
            res.add(cur);
        }
    }
    return res.toArray(new int[0][]);
}

public static int[][] insert(int[][] intervals, int[] newInterval) {
    List<int[]> res = new ArrayList<>();
    int i = 0, n = intervals.length;
    while (i < n && intervals[i][1] < newInterval[0]) res.add(intervals[i++]);
    while (i < n && intervals[i][0] <= newInterval[1]) {
        newInterval[0] = Math.min(newInterval[0], intervals[i][0]);
        newInterval[1] = Math.max(newInterval[1], intervals[i][1]);
        i++;
    }
    res.add(newInterval);
    while (i < n) res.add(intervals[i++]);
    return res.toArray(new int[0][]);
}
```

**Complexity.** Merge: O(n log n) for the sort. Insert: O(n).
