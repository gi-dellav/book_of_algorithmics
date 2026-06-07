# Locality of Reference

Locality is the property that makes caches work. A program exhibits **temporal locality** if it accesses the same data multiple times in a short period. It exhibits **spatial locality** if it accesses data near other recently-accessed data. Both are essential for performance.

## Spatial Locality

Spatial locality is about using the *entire* cache line. A cache line is 64 bytes on x86 — 16 ints, 8 doubles, or 64 chars. When you load one element, you get its 15 neighbors for free (they're in the same cache line). The next 15 accesses to those neighbors cost zero additional I/O.

**Good spatial locality**:
```c
for (int i = 0; i < n; i++)
    sum += a[i];  // Sequential → every element after the first is a cache hit
```

**Poor spatial locality**:
```c
for (int j = 0; j < n; j++)
    for (int i = 0; i < n; i++)
        sum += a[i * stride];  // Strided → each access may be to a different cache line
```

When `stride = n` (column access in a row-major matrix), every access is to a different cache line. At stride = 64 bytes (exactly one cache line), every access is a cache miss — worst-case spatial locality.

## Temporal Locality

Temporal locality is about reusing data before it's evicted from cache.

**Good temporal locality**:
```c
for (int k = 0; k < n; k++)      // K is the inner loop
    for (int i = 0; i < n; i++)
        C[i][j] += A[i][k] * B[k][j];  // B[k][j] is reused n times (temporal in k loop)
```

**Poor temporal locality**:
```c
for (int i = 0; i < n; i++)      // I is the outer loop
    for (int j = 0; j < n; j++)
        C[i][j] += A[i][k] * B[k][j];  // Each element accessed once, never reused in cache
```

The second version streams through data but doesn't reuse anything. If the working set exceeds cache size, every access in the inner loop is a miss.

## Case Study: Merge Sort

### Depth-First Merge Sort

```c
void mergesort(int *a, int *buf, int lo, int hi) {
    if (hi - lo <= 1) return;
    int mid = (lo + hi) / 2;
    mergesort(a, buf, lo, mid);
    mergesort(a, buf, mid, hi);
    merge(a, buf, lo, mid, hi);
}
```

The recursion sorts the left half completely, then the right half completely, then merges. The left half fits in cache (for sufficiently small subproblems), and the merge is a sequential scan. This is cache-efficient: the working set at each level is the size of the current subarray.

### Breadth-First Merge Sort

An alternative: sort all size-2 chunks, then merge into size-4 chunks, then size-8, etc. This is how external merge sort works. It's more cache-friendly for very large arrays because each pass does a sequential scan of the entire array (minimizing cache misses per element) rather than recursively descending into subproblems that eventually don't fit in cache.

The crossover between depth-first and breadth-first depends on the ratio of array size to cache size. For in-memory sorting, depth-first (standard mergesort) is fine up to a few MB. For external sorting, breadth-first (multiway merge sort) is mandatory.

## Dynamic Programming: Knapsack

The standard DP for 0/1 knapsack:

```c
for (int i = 1; i <= n; i++)
    for (int w = 0; w <= W; w++)
        dp[i][w] = max(dp[i-1][w], dp[i-1][w - weight[i]] + value[i]);
```

Row `i` depends only on row `i-1`. Only two rows need to be in cache simultaneously — excellent temporal locality. The inner loop is sequential in `w` — excellent spatial locality. This is why DP fits in cache easily despite the O(nW) theoretical complexity.

But if the DP order is reversed (column-major), the spatial locality disappears: `dp[i][w]` for consecutive w values are now separated by n elements, striding across cache lines. The same algorithm, simply reordered, can be 10–100× slower.

## Sparse Table for RMQ

A sparse table answers range minimum queries in O(1) after O(n log n) preprocessing:

```
st[k][i] = min(st[k-1][i], st[k-1][i + 2^{k-1}])
```

The naive layout (`int st[LOG][N]`) has `st[k][i]` and `st[k][i+1]` adjacent — good spatial locality within one level. But `st[k-1][i]` and `st[k-1][i + 2^{k-1}]` are far apart — poor spatial locality when building level k from level k-1.

Alternative layout: interleave the levels. Store `st[0][0], st[1][0], st[2][0], ..., st[0][1], st[1][1], ...`. This puts consecutive levels of the same index in the same cache line. Building the table now has excellent spatial locality. Querying the table (two accesses to possibly different levels) is neutral or slightly worse.

The right layout depends on whether build time or query time dominates. The general principle: **organize data for how it's accessed, not for how it's conceptually structured.**

## AoS vs. SoA (Row vs. Columnar Layout)

```c
// Array of Structures (AoS)
struct Point { float x, y, z; };
Point points[N];

// Structure of Arrays (SoA)
struct Points { float x[N], y[N], z[N]; };
```

If you're iterating over all points and accessing all three coordinates, both layouts are fine (both stream sequentially). But if you're computing `magnitude = sqrt(x² + y² + z²)`, AoS loads x, y, z from one cache line (good — all needed data is together). If you're computing `sum(x)`, SoA streams through x values without loading y and z (good — no wasted bandwidth).

**Rule**: Use AoS when each access touches all (or most) fields. Use SoA when hot loops touch a subset of fields. The `cpu-cache/aos-soa.md` article has benchmarks showing 3–5× differences in some cases.

This is the same design choice as row-major vs. column-major database storage. OLTP (transaction processing, accessing entire rows) → row-major. OLAP (analytical queries, scanning columns) → column-major.
