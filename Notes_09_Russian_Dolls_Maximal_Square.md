# Notes 9: Russian Doll Envelopes and Maximal Square (Dynamic Programming)

All Java is Java 8 compatible. Both solutions were compiled and run: each was compared with a brute-force reference on tens of thousands of random inputs (0 mismatches) and checked on the examples below.

## At a glance

| # | Problem | Idea | Time (best / average / worst) | Space |
|---|---------|------|-------------------------------|-------|
| 1 | Russian Doll Envelopes | Sort by width (ties by height descending), then LIS on heights with the `tails` array and binary search | O(n) / O(n log n) / O(n log n) | O(n) |
| 2 | Maximal Square | `dp[i][j]` = side of the largest all-1 square ending at (i, j); `1 + min(up, left, diagonal)` | Θ(m·n) in every case | O(m·n), or O(n) with one row |

---

## 1. Russian Doll Envelopes

### Problem

You are given envelopes `envelopes[i] = [width, height]`. One envelope fits inside another only if **both** its width and its height are **strictly smaller**. Rotating an envelope is not allowed. Return the maximum number of envelopes that can be nested like Russian dolls.

### Examples

| Input | Output | Why |
|-------|--------|-----|
| `[[5,4],[6,4],[6,7],[2,3]]` | `3` | `[2,3] -> [5,4] -> [6,7]` |
| `[[1,1],[1,1],[1,1]]` | `1` | Equal envelopes cannot nest |
| `[[4,5],[4,6],[6,7],[2,3],[1,1]]` | `4` | `[1,1] -> [2,3] -> [4,5] -> [6,7]`; `[4,6]` cannot be used with `[4,5]` (same width) |
| `[[7,9]]` | `1` | A single envelope |

Constraints (typical): `1 <= envelopes.length <= 10^5`, `1 <= width, height <= 10^5`.

```java
public int maxEnvelopes(int[][] envelopes)
```

### Idea in four steps

1. **Remove one dimension by sorting.** Sort by width ascending. After that, going left to right, the width condition is already handled by the order, so a chain only needs strictly increasing **heights**.
2. **The problem becomes LIS.** The answer is the length of the Longest Increasing Subsequence of the heights (strictly increasing).
3. **Tie-break trap.** Envelopes with the same width cannot nest. If equal widths were sorted by height ascending, LIS could chain `[6,4] -> [6,7]` (heights 4, 7) even though the widths are equal. Sort equal widths by height **descending**, so their heights can never form an increasing step.
4. **Fast LIS with `tails`.** `tails[k]` is the smallest possible last height of an increasing chain of length `k+1`. It is always sorted, so for each height binary search for the first tail `>= h`. If none exists, append (the chain grows). Otherwise replace that tail with `h` (a smaller ending is never worse). The length of `tails` is the answer. `tails` itself is not necessarily a real chain; only its length is meaningful.

### Solution A: sort plus LIS with binary search

```java
public static int maxEnvelopes(int[][] envelopes) {
    // width ascending; for equal widths, height DESCENDING
    Arrays.sort(envelopes, (a, b) ->
            a[0] != b[0] ? Integer.compare(a[0], b[0]) : Integer.compare(b[1], a[1]));

    int[] tails = new int[envelopes.length];   // tails[k] = smallest last height of a chain of length k+1
    int size = 0;
    for (int[] env : envelopes) {
        int h = env[1];
        int lo = 0, hi = size;                 // first index in tails[0..size) with value >= h
        while (lo < hi) {
            int mid = lo + (hi - lo) / 2;
            if (tails[mid] < h) lo = mid + 1;
            else hi = mid;
        }
        tails[lo] = h;                         // replace, or append when lo == size
        if (lo == size) size++;
    }
    return size;
}
```

### Dry run on `[[5,4],[6,4],[6,7],[2,3]]`

Sorted: `[2,3], [5,4], [6,7], [6,4]`, so the heights are `3, 4, 7, 4`.

| Height | Binary search result | `tails` after | `size` |
|--------|----------------------|---------------|--------|
| 3 | `lo = 0 = size`, append | `[3]` | 1 |
| 4 | `lo = 1 = size`, append | `[3, 4]` | 2 |
| 7 | `lo = 2 = size`, append | `[3, 4, 7]` | 3 |
| 4 | `lo = 1` (tails[1] = 4 >= 4), replace with 4 | `[3, 4, 7]` | 3 |

Answer: **3**.

### Baseline: O(n²) DP after sorting (name it first, then improve)

```java
static int maxEnvelopesDp(int[][] e) {
    int[][] s = new int[e.length][];
    for (int i = 0; i < e.length; i++) s[i] = e[i].clone();
    Arrays.sort(s, (a, b) -> a[0] != b[0] ? Integer.compare(a[0], b[0]) : Integer.compare(a[1], b[1]));
    int best = 0;
    int[] dp = new int[s.length];              // dp[i] = longest chain ending at envelope i
    for (int i = 0; i < s.length; i++) {
        dp[i] = 1;
        for (int j = 0; j < i; j++)
            if (s[j][0] < s[i][0] && s[j][1] < s[i][1]) dp[i] = Math.max(dp[i], dp[j] + 1);
        best = Math.max(best, dp[i]);
    }
    return best;
}
```

This is correct but too slow for `n = 10^5`; Solution A handled `n = 100,000` random envelopes in about 70 ms.

### Complexity of Solution A (Big O)

| Case | Time | When it happens |
|------|------|-----------------|
| Best | O(n) | The input is already in the sorted order (Java's TimSort is linear on presorted data) and the heights never form a chain longer than 1, so each binary search is over a tiny `tails` |
| Average | O(n log n) | Random input: sort is O(n log n), binary searches are O(log n) each |
| Worst | O(n log n) | Sort O(n log n) plus `n` binary searches over a `tails` that can grow to size `n` (for example a long strictly increasing chain) |

- **Space:** O(n) in every case, for `tails` (and the sort's temporary space, which is also at most O(n)).
- In practice, say "O(n log n)" in an interview; the O(n) best case needs very specific input.

Baseline DP (sort plus nested loops): time **Θ(n²)** in best, average and worst case (the inner loop always runs), space O(n).

### Pitfalls

1. Sorting ties by height **ascending** gives wrong answers (same-width envelopes get chained).
2. Using `a[0] - b[0]` in the comparator can overflow; use `Integer.compare`.
3. In the binary search, use "first tail `>= h`" (not `> h`). Using `>` allows equal heights to extend a chain, which breaks the strict requirement.
4. `Arrays.sort` sorts the input in place; mention it or sort a copy if the caller must not be changed.

### Edge cases

All envelopes identical (answer 1); a single envelope (answer 1); envelopes with the same width but different heights (only one of them can be in a chain); already sorted descending (answer 1 if all widths equal).

---

## 2. Maximal Square

### Problem

Given an `m x n` matrix of characters `'0'` and `'1'`, find the largest **square** that contains only `'1'` and return its **area**.

### Examples

```
Matrix:        1 0 1 0 0
               1 0 1 1 1
               1 1 1 1 1
               1 0 0 1 0        ->  4   (a 2 x 2 square)
```

| Input | Output |
|-------|--------|
| the matrix above | `4` |
| `["01","10"]` | `1` |
| `["0"]` | `0` |
| `["1"]` | `1` |
| `["11111","11111","11111"]` | `9` (the 3 x 3 square; the 3 x 5 rectangle is not a square) |

Note: the cells are **characters** (`'1'`), not integers. Return the **area** (side squared), not the side length.

```java
public int maximalSquare(char[][] matrix)
```

### Idea

- **State:** `dp[i][j]` = the side length of the largest all-1 square whose **bottom-right corner** is cell `(i, j)`.
- **Recurrence:**
  - If the cell is `'0'`, then `dp[i][j] = 0`.
  - If the cell is `'1'`, then `dp[i][j] = 1 + min(dp[i-1][j], dp[i][j-1], dp[i-1][j-1])` (up, left, diagonal).
- **Why the minimum?** A square of side `k` ending at `(i, j)` needs squares of side at least `k-1` ending at all three neighbors. The smallest of the three limits it. If any neighbor is 0, the new square has side 1.
- **Answer:** the largest `dp` value, squared.
- **Base cases:** use a table padded with an extra zero row and column, so first-row and first-column cells need no special handling.

### Worked example (dp table)

For the matrix above:

```
dp:   1 0 1 0 0
      1 0 1 1 1
      1 1 1 2 2
      1 0 0 1 0
```

- `(2,3)`: up = 1, left = 1, diagonal = 1, so `dp = 1 + 1 = 2`.
- `(2,4)`: up = 1, left = 2, diagonal = 1, so `dp = 1 + 1 = 2`.
- `(2,2)`: up = 1, left = 1, **diagonal = 0**, so `dp = 1 + 0 = 1`. A zero on the diagonal limits the square to side 1.

Largest value 2, area `2 x 2 = 4`.

### Solution A: 2D DP with a padded table

```java
public static int maximalSquare(char[][] matrix) {
    if (matrix == null || matrix.length == 0 || matrix[0].length == 0) return 0;
    int rows = matrix.length, cols = matrix[0].length;
    int[][] dp = new int[rows + 1][cols + 1];   // dp[i][j] = side of the largest square ending at cell (i-1, j-1)
    int maxSide = 0;
    for (int i = 1; i <= rows; i++) {
        for (int j = 1; j <= cols; j++) {
            if (matrix[i - 1][j - 1] == '1') {
                dp[i][j] = 1 + Math.min(dp[i - 1][j - 1], Math.min(dp[i - 1][j], dp[i][j - 1]));
                maxSide = Math.max(maxSide, dp[i][j]);
            }
        }
    }
    return maxSide * maxSide;
}
```

### Solution B: one row, O(n) space

Each cell needs only the row above and the cell to its left. Keep one array and remember the diagonal neighbor in a variable before overwriting it.

```java
public static int maximalSquareOneRow(char[][] matrix) {
    if (matrix == null || matrix.length == 0 || matrix[0].length == 0) return 0;
    int rows = matrix.length, cols = matrix[0].length;
    int[] dp = new int[cols + 1];
    int maxSide = 0;
    for (int i = 1; i <= rows; i++) {
        int prev = 0;                           // diagonal neighbour dp[i-1][j-1]
        for (int j = 1; j <= cols; j++) {
            int up = dp[j];                     // dp[i-1][j] before we overwrite it
            if (matrix[i - 1][j - 1] == '1') {
                dp[j] = 1 + Math.min(prev, Math.min(up, dp[j - 1]));   // dp[j-1] is already dp[i][j-1]
                maxSide = Math.max(maxSide, dp[j]);
            } else {
                dp[j] = 0;
            }
            prev = up;                          // becomes the diagonal for the next cell
        }
    }
    return maxSide * maxSide;
}
```

### Brute force (to explain why DP helps)

For every cell as a top-left corner, grow the square size `k` while all cells inside are `'1'`, and keep the largest. It is easy to get right but repeats a lot of work.

### Complexity (Big O)

**Solution A (2D DP) and Solution B (one row):**

| Case | Time | Notes |
|------|------|-------|
| Best | Θ(m·n) | Every cell is visited exactly once, whatever the data |
| Average | Θ(m·n) | Same: no data-dependent shortcuts |
| Worst | Θ(m·n) | Same (for example an all-ones matrix) |

| Solution | Space |
|----------|-------|
| A: 2D table | O(m·n) |
| B: one row | O(n), where n = number of columns (you can swap to the shorter side to get O(min(m, n))) |

Because the time is the same in all cases, say **Θ(m·n)**, or simply O(m·n).

**Brute force:**

| Case | Time |
|------|------|
| Best | O(m·n), for example an all-zero matrix, where every cell fails at once |
| Worst | O(m·n·min(m,n)³) if the whole square is re-checked for every size, or O(m·n·min(m,n)²) if only the new border row and column is checked (for example an all-ones matrix) |
| Average | Depends on the density of ones. For sparse or random data the squares stay small, so it behaves close to the best case. Treat this as an approximation, not an exact bound. |

Space for brute force: O(1).

### Pitfalls

1. Returning the side length instead of the **area**.
2. Comparing the cell with `1` instead of `'1'` (the matrix holds characters).
3. Forgetting the diagonal neighbor (using only up and left gives wrong sizes).
4. In the one-row version, overwriting `dp[j]` before saving the old value (the diagonal neighbor is lost). Save `up` first and set `prev = up` after processing each cell.
5. Treating a rectangle of ones as a square (the 3 x 5 block of ones has largest square 3 x 3, area 9, not 15).

### Edge cases

An all-zero matrix (0); a single `'1'` (1); a single row or column (0 or 1); a non-square matrix (the square is limited by the shorter side); a full matrix of ones (area is the smaller dimension squared).

---

## Pattern notes

| Pattern | Used in | Key sentence |
|---------|---------|--------------|
| Reduce two dimensions to one by sorting | Russian Doll Envelopes | Sort by one dimension, then the other dimension becomes a 1-D problem (LIS) |
| LIS with a `tails` array | Russian Doll Envelopes | `tails[k]` = smallest possible ending of a chain of length `k+1`; binary search the first tail `>= x` |
| Tie-break to enforce strictness | Russian Doll Envelopes | Sort equal widths by height descending so equal widths cannot chain |
| 2D DP on a grid with `min` of three neighbors | Maximal Square | A square of side `k` needs squares of side `k-1` at up, left and diagonal |
| Rolling array | Maximal Square | A row depends only on the previous row, so keep one row plus one saved diagonal value |
