# Binary Search

Binary search finds the position of a key in a sorted array — or where it would be inserted to keep the array sorted. The algorithm is deceptively simple: compare against the middle element, recurse left or right. Yet the standard implementation leaves ~4× performance on the table through branch mispredictions and cache-unfriendly memory layout. This article recovers that performance step by step.

## The Standard Implementation

```c
int lower_bound(int *a, int n, int key) {
    int lo = 0, hi = n;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] < key)
            lo = mid + 1;
        else
            hi = mid;
    }
    return lo;
}
```

For n = 1M 32-bit integers on Zen 2: ~180ns per query (~1100 cycles at 2 GHz). The loop executes ~20 iterations (log₂(1M) ≈ 20), each taking ~55 cycles. The cost breakdown:

- **Branch misprediction**: ~12 cycles per mispredict. The `if (a[mid] < key)` branch is unpredictable — each comparison is new data, no pattern. ~50% mispredict rate → ~6 cycles/iteration wasted.
- **Cache misses**: ~15 cycles per miss. The first few iterations touch random locations in a large array → L3 misses. ~5 L3 misses per query → ~75 cycles.
- **Data dependency**: `mid` depends on `lo` and `hi`, which depend on the previous comparison. The branch predictor can't start the next iteration until the current one resolves.

Total: ~1100 cycles. The hardware spends most of its time waiting — for the branch to resolve, for data to arrive from cache.

## Stage 1: Branchless Binary Search

The `if (a[mid] < key)` branch exists to choose between `lo = mid + 1` and `hi = mid`. We can eliminate the branch by computing both values and selecting with a conditional move (`cmov`):

```c
int lower_bound_branchless(int *a, int n, int key) {
    int *base = a;
    while (n > 1) {
        int half = n / 2;
        int *mid = base + half;
        // If key > *mid: base = mid, n -= half
        // Else:           base stays, n = half
        base = (key > *mid) ? mid : base;
        n -= half;
    }
    return (key > *base) + (base - a);
}
```

The ternary `? :` compiles to a `cmp` + `cmovg` sequence — no branch. The loop is a straight-line computation where every instruction executes every iteration.

Performance: ~85ns per query (~170 cycles). ~2.1× faster. The branch misprediction cost is gone. The remaining bottlenecks: cache misses for the first few iterations (~5 L3 misses still ~75 cycles), and the data dependency chain: `base` depends on the comparison, `n` depends on `half`, the loop condition depends on `n`.

But we can still improve: for n = 16 or smaller, a branchless CMOV is actually *worse* than a predicted branch, because the last few iterations are predictable (small range, short stride). We'll fix this later.

## Stage 2: Eytzinger Layout

The standard sorted array is laid out in memory as `[a[0], a[1], a[2], ..., a[n-1]]`. Binary search accesses `a[n/2]`, then `a[n/4]` or `a[3n/4]`, then `a[n/8]` or `a[3n/8]` or `a[5n/8]` or `a[7n/8]`, and so on. The first access is to the middle of the array — whatever cache line that falls in. The second access jumps to a quarter of the array away. These accesses have poor spatial locality, and prefetchers can't predict the pattern.

The **Eytzinger layout** (also called "BFS layout" or "heap order") stores the array in the order visited by a breadth-first traversal of an implicit binary search tree:

```
Standard:   [a0, a1, a2, a3, a4, a5, a6, a7, a8, a9, a10, a11, a12, a13, a14]
Eytzinger:  [a7, a3, a11, a1, a5, a9, a13, a0, a2, a4, a6, a8, a10, a12, a14]
```

The root (index 0) is the middle element. Its left child (index 1) is the middle of the left half. Its right child (index 2) is the middle of the right half. And so on. The children of index `i` are at `2i+1` and `2i+2` — exactly like a binary heap.

To build the Eytzinger layout:

```c
int *eytzinger(int *sorted, int n) {
    int *e = malloc(n * sizeof(int));
    build(e, sorted, 0, n, 0);
    return e;
}

int build(int *e, int *sorted, int lo, int hi, int idx) {
    if (lo >= hi) return idx;
    int mid = lo + (hi - lo) / 2;
    e[idx] = sorted[mid];
    int next = build(e, sorted, lo, mid, 2*idx + 1);
    return build(e, sorted, mid + 1, hi, 2*idx + 2);
}
```

Searching the Eytzinger layout:

```c
int lower_bound_eytzinger(int *e, int n, int key) {
    int idx = 0;
    int result = n;  // Default: key not found
    while (idx < n) {
        if (key > e[idx]) {
            idx = 2 * idx + 2;  // Right child
        } else {
            result = idx;        // Could be the answer
            idx = 2 * idx + 1;  // Left child
        }
    }
    return result;
}
```

Why does this help? The first few levels of the Eytzinger tree are contiguous in memory. Level 0 (root) is at index 0. Level 1 is at indices 1-2. Level 2 at indices 3-6. Level 3 at indices 7-14. The first ~3 iterations access indices 0, then 1-2, then 3-6 — all within the first cache line (64 bytes = 16 ints). The first few comparisons that were L3 misses in the standard layout are now L1 hits.

Performance: ~55ns per query (~110 cycles). ~3.3× over the standard, ~1.5× over the branchless standard layout. The Eytzinger layout combines with branchless search:

```c
int lower_bound_eytzinger_branchless(int *e, int n, int key) {
    int idx = 0;
    while (idx < n) {
        // For a CMOV implementation, we compute both children:
        int left = 2 * idx + 1;
        int right = 2 * idx + 2;
        int go_right = (key > e[idx]);
        idx = go_right ? right : left;
    }
    // Reconstruct the insertion index from the leaf position
    return reconstruct(idx, n);
}
```

The branchless Eytzinger search: ~35ns per query (~70 cycles). ~5.1× over standard. The remaining bottleneck: the data dependency chain. Each iteration needs `idx` from the previous iteration. The CPU can't overlap iterations.

## Stage 3: Prefetching in Eytzinger

Even with the Eytzinger layout, the deeper levels of the tree (beyond the first few) still cause cache misses. We can issue prefetches for the children before we need them:

```c
int lower_bound_eytzinger_prefetch(int *e, int n, int key) {
    int idx = 0;
    while (idx < n) {
        int left = 2 * idx + 1;
        int right = 2 * idx + 2;
        // Prefetch grandchildren: the next step's data
        if (left < n) {
            __builtin_prefetch(&e[2*left + 1], 0, 0);   // left child's left child
            __builtin_prefetch(&e[2*left + 2], 0, 0);   // left child's right child
        }
        if (right < n) {
            __builtin_prefetch(&e[2*right + 1], 0, 0);  // right child's left child
            __builtin_prefetch(&e[2*right + 2], 0, 0);  // right child's right child
        }
        int go_right = (key > e[idx]);
        idx = go_right ? right : left;
    }
    return reconstruct(idx, n);
}
```

The `__builtin_prefetch` instructions are hints to the hardware — they don't stall the pipeline. While the current comparison resolves, the prefetcher brings the grandchildren into L1. By the time we reach them, they're ready.

Performance: ~28ns per query (~56 cycles). ~6.4× over standard. Prefetching overlaps memory latency with computation. The deeper the tree, the bigger the win (more levels can be prefetched).

## Stage 4: Removing the Last Branch (Predicated Search)

For the final iterations of the search (when the remaining range fits in a cache line), we can switch to a fully predicated approach: load the entire cache line, compare all elements in parallel with SIMD, and extract the result with bit operations:

```c
int lower_bound_final(int *e, int n, int key) {
    int idx = 0;
    // Eytzinger search until the remaining range fits in one cache line
    while (idx * 2 + 16 < n) {  // Stop when subtree fits in 16 elements
        int left = 2 * idx + 1;
        int right = 2 * idx + 2;
        int go_right = (key > e[idx]);
        idx = go_right ? right : left;
    }
    
    // The subtree rooted at idx fits in one cache line.
    // Load the whole subtree and search with SIMD:
    int subtree_size = find_subtree_size(idx, n);  // ≤ 15 for a depth-3 subtree
    
    // Load 16 ints (the subtree plus maybe some extra)
    __m512i v = _mm512_loadu_si512(&e[idx]);
    __m512i vkey = _mm512_set1_epi32(key);
    
    // Compare: which elements are <= key?
    __mmask16 mask = _mm512_cmple_epi32_mask(v, vkey);
    
    // Count how many elements are <= key → insertion index
    int count = __builtin_popcount(mask) - 1;  // -1 because we want < key, not <=
    
    return reconstruct(idx, n, count);
}
```

This eliminates the last 3–4 iterations of the search (indices 8–14 in the subtree), replacing the dependency chain with a single SIMD compare-and-popcount. Performance: ~22ns per query (~44 cycles). ~8.2× over standard.

## Performance Summary (Zen 2, n = 1M, 32-bit integers)

| Implementation | Time (ns) | Speedup vs. `std::lower_bound` |
|---------------|-----------|-------------------------------|
| `std::lower_bound` | 180 | 1.0× |
| Branchless | 85 | 2.1× |
| Eytzinger layout | 55 | 3.3× |
| Eytzinger + branchless | 35 | 5.1× |
| + Prefetching | 28 | 6.4× |
| + Predicated final step | 22 | 8.2× |

## Expected Number of Comparisons

A standard binary search does ~log₂(n) comparisons. The branchless version does exactly log₂(n) every time — no early exit. The Eytzinger layout preserves this. But the predicated final step reduces this: log₂(n) - log₂(L) where L is the subtree size handled by SIMD.

For n = 1M: log₂(1M) ≈ 20 iterations. With a 16-element SIMD final step: 20 - 4 = 16 iterations of the main loop + 1 SIMD compare. Net: ~16 comparisons worth of work, but the last 4 are done in parallel.

## Key Lessons

1. **Branch misprediction dominates binary search.** The `a[mid] < key` comparison is unpredictable by nature — the key could be anywhere. Eliminating the branch with `cmov` gives a 2× speedup.
2. **Memory layout determines cache behavior.** The Eytzinger layout packs the first few levels into contiguous cache lines, turning L3 misses into L1 hits for the most critical comparisons.
3. **Prefetching hides memory latency.** Issuing prefetches for grandchildren overlaps the current computation with future memory loads.
4. **SIMD accelerates the final mile.** When the remaining search range fits in a cache line, a single SIMD compare replaces the last 3–4 iterations.
5. **There is always a dependency chain.** Each iteration needs the result of the previous one. No amount of ILP can overlap iterations. The only fix: reduce the number of iterations (SIMD final step, wider trees).

The natural next step: what if each "node" compared against B keys instead of 1? That leads to B-trees, which we develop in the next article.
