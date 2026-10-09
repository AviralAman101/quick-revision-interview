# Notes 2 of 6: Questions 6 to 10

Two pointers, prefix sums, heaps. Java 8 compatible.

> Reconstructed from the topic list you gave me (the original session statements and your submitted code were not retained). The statements and solutions are standard, clean reference versions, not your exact submissions.

---

## Q6. Trapping Rain Water

**Part A.** `height[i]` is the height of a bar of width 1. Return how much water is trapped after raining. Use two pointers with O(1) space.

```
[0,1,0,2,1,0,1,3,2,1,2,1] -> 6
[4,2,0,3,2,5]             -> 9
[1,2,3]                   -> 0
```

**Part B.** Solve the same problem with prefix-maximum and suffix-maximum arrays.

**Idea.** Water above bar `i` is `min(maxLeft[i], maxRight[i]) - height[i]`. Two pointers avoid the arrays: always move the side with the smaller current bar, because that side's water level is limited by its own running maximum.

```java
public static int trap(int[] height) {
    int l = 0, r = height.length - 1;
    int leftMax = 0, rightMax = 0, water = 0;
    while (l < r) {
        if (height[l] < height[r]) {
            if (height[l] >= leftMax) leftMax = height[l];
            else water += leftMax - height[l];
            l++;
        } else {
            if (height[r] >= rightMax) rightMax = height[r];
            else water += rightMax - height[r];
            r--;
        }
    }
    return water;
}

public static int trapPrefixSuffix(int[] height) {
    int n = height.length;
    if (n == 0) return 0;
    int[] left = new int[n], right = new int[n];
    left[0] = height[0];
    for (int i = 1; i < n; i++) left[i] = Math.max(left[i - 1], height[i]);
    right[n - 1] = height[n - 1];
    for (int i = n - 2; i >= 0; i--) right[i] = Math.max(right[i + 1], height[i]);
    int water = 0;
    for (int i = 0; i < n; i++) water += Math.min(left[i], right[i]) - height[i];
    return water;
}
```

**Complexity.** Part A: O(n) time, O(1) space. Part B: O(n) time, O(n) space.

---

## Q7. Product of Array Except Self

**Part A.** Return `answer` where `answer[i]` is the product of all elements except `nums[i]`. Do not use division. Use prefix and suffix product arrays.

```
[1,2,3,4]    -> [24,12,8,6]
[-1,1,0,-3,3] -> [0,0,9,0,0]
[2,3]        -> [3,2]
```

**Part B.** Solve it with O(1) extra space (the output array does not count).

**Idea.** `answer[i] = (product of everything left of i) x (product of everything right of i)`. Fill the output with left products, then multiply in the right products on a backward pass using one running variable.

```java
public static int[] productExceptSelf(int[] nums) {
    int n = nums.length;
    int[] left = new int[n], right = new int[n], res = new int[n];
    left[0] = 1;
    for (int i = 1; i < n; i++) left[i] = left[i - 1] * nums[i - 1];
    right[n - 1] = 1;
    for (int i = n - 2; i >= 0; i--) right[i] = right[i + 1] * nums[i + 1];
    for (int i = 0; i < n; i++) res[i] = left[i] * right[i];
    return res;
}

public static int[] productExceptSelfO1(int[] nums) {
    int n = nums.length;
    int[] res = new int[n];
    res[0] = 1;
    for (int i = 1; i < n; i++) res[i] = res[i - 1] * nums[i - 1];
    int suffix = 1;
    for (int i = n - 1; i >= 0; i--) {
        res[i] *= suffix;
        suffix *= nums[i];
    }
    return res;
}
```

**Complexity.** O(n) time. Space O(n) for Part A, O(1) extra for Part B.

---

## Q8. Subarray Sum Equals K

**Part A.** Return the number of contiguous subarrays whose sum equals `k`. The array can contain negatives.

```
[1,1,1], k=2   -> 2
[1,2,3], k=3   -> 2   ([1,2] and [3])
[1,-1,0], k=0  -> 3
```

**Part B.** Return the length of the longest contiguous subarray whose sum equals `k` (or 0 if none).

```
[1,-1,5,-2,3], k=3 -> 4
[-2,-1,2,1],   k=1 -> 2
[1,2],         k=9 -> 0
```

**Idea.** Prefix sums with a hash map. A subarray ending at `i` sums to `k` exactly when some earlier prefix equals `sum - k`. For counting, store how many times each prefix occurred. For the longest, store only the first index of each prefix.

```java
public static int subarraySum(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    freq.put(0, 1);
    int sum = 0, count = 0;
    for (int x : nums) {
        sum += x;
        count += freq.getOrDefault(sum - k, 0);
        freq.merge(sum, 1, Integer::sum);
    }
    return count;
}

public static int longestSubarrayWithSumK(int[] nums, int k) {
    Map<Integer, Integer> firstIndex = new HashMap<>();
    firstIndex.put(0, -1);
    int sum = 0, best = 0;
    for (int i = 0; i < nums.length; i++) {
        sum += nums[i];
        Integer j = firstIndex.get(sum - k);
        if (j != null) best = Math.max(best, i - j);
        firstIndex.putIfAbsent(sum, i);
    }
    return best;
}
```

**Complexity.** O(n) time and space.

---

## Q9. Top K Frequent Elements

**Part A.** Return the `k` most frequent elements in any order. Use a hash map plus a min-heap of size `k`.

```
[1,1,1,2,2,3], k=2 -> [1,2]
[1], k=1           -> [1]
[4,4,4,5,5,6,6,6,6], k=1 -> [6]
```

**Part B.** Solve it in O(n) with bucket sort.

**Idea.** Count frequencies. Heap version: keep the `k` most frequent in a min-heap by count and pop the smallest when the size exceeds `k`. Bucket version: index a list by frequency (frequency is at most `n`) and read buckets from the highest.

```java
public static int[] topKFrequent(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int x : nums) freq.merge(x, 1, Integer::sum);
    PriorityQueue<Integer> pq = new PriorityQueue<>((a, b) -> Integer.compare(freq.get(a), freq.get(b)));
    for (int key : freq.keySet()) {
        pq.offer(key);
        if (pq.size() > k) pq.poll();
    }
    int[] res = new int[k];
    for (int i = k - 1; i >= 0; i--) res[i] = pq.poll();
    return res;
}

public static int[] topKFrequentBucket(int[] nums, int k) {
    Map<Integer, Integer> freq = new HashMap<>();
    for (int x : nums) freq.merge(x, 1, Integer::sum);
    List<List<Integer>> buckets = new ArrayList<>();
    for (int i = 0; i <= nums.length; i++) buckets.add(new ArrayList<>());
    for (Map.Entry<Integer, Integer> e : freq.entrySet()) buckets.get(e.getValue()).add(e.getKey());
    int[] res = new int[k];
    int idx = 0;
    for (int f = nums.length; f >= 1 && idx < k; f--) {
        for (int x : buckets.get(f)) {
            if (idx == k) break;
            res[idx++] = x;
        }
    }
    return res;
}
```

**Complexity.** Heap: O(n log k). Bucket: O(n).

---

## Q10. K Closest Points to Origin (A) and Kth Largest Element (B)

**Part A.** Return the `k` points closest to the origin (Euclidean distance, any order).

```
points=[[1,3],[-2,2]], k=1          -> [[-2,2]]
points=[[3,3],[5,-1],[-2,4]], k=2   -> [[3,3],[-2,4]]
```

**Part B.** Return the `k`-th largest element of an array (counting duplicates).

```
[3,2,1,5,6,4], k=2          -> 5
[3,2,3,1,2,4,5,5,6], k=4    -> 4
```

**Idea.** Keep a heap of size `k`. For closest points, use a **max**-heap by squared distance and evict the farthest when the size passes `k`. For the `k`-th largest, use a **min**-heap of size `k`: its top is the answer.

```java
public static int[][] kClosest(int[][] points, int k) {
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) ->
        Integer.compare(b[0] * b[0] + b[1] * b[1], a[0] * a[0] + a[1] * a[1]));
    for (int[] p : points) {
        pq.offer(p);
        if (pq.size() > k) pq.poll();             // drop the farthest
    }
    int[][] res = new int[k][];
    for (int i = 0; i < k; i++) res[i] = pq.poll();
    return res;
}

public static int findKthLargest(int[] nums, int k) {
    PriorityQueue<Integer> pq = new PriorityQueue<>();
    for (int x : nums) {
        pq.offer(x);
        if (pq.size() > k) pq.poll();
    }
    return pq.peek();
}
```

**Complexity.** O(n log k) time, O(k) space.
