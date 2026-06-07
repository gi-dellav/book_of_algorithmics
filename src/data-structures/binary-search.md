# Binary Search

Binary search finds the position of a key in a sorted array — or where it would be inserted to keep the array sorted. The algorithm is deceptively simple: compare against the middle element, recurse left or right. Yet the standard implementation leaves ~4× performance on the table through branch mispredictions and cache-unfriendly memory layout. This article recovers that performance step by step.

## The Standard Implementation

```rust
fn lower_bound(a: *const i32, n: usize, key: i32) -> usize {
    let mut lo: usize = 0;
    let mut hi: usize = n;
    while lo < hi {
        let mid = lo + (hi - lo) / 2;
        if unsafe { *a.add(mid) } < key {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    lo
}
```

For n = 1M 32-bit integers on Zen 2: ~180ns per query (~1100 cycles at 2 GHz). The loop executes ~20 iterations (log₂(1M) ≈ 20), each taking ~55 cycles. The cost breakdown:

- **Branch misprediction**: ~12 cycles per mispredict. The `if (a[mid] < key)` branch is unpredictable — each comparison is new data, no pattern. ~50% mispredict rate → ~6 cycles/iteration wasted.
- **Cache misses**: ~15 cycles per miss. The first few iterations touch random locations in a large array → L3 misses. ~5 L3 misses per query → ~75 cycles.
- **Data dependency**: `mid` depends on `lo` and `hi`, which depend on the previous comparison. The branch predictor can't start the next iteration until the current one resolves.

Total: ~1100 cycles. The hardware spends most of its time waiting — for the branch to resolve, for data to arrive from cache.

## Stage 1: Branchless Binary Search

The `if (a[mid] < key)` branch exists to choose between `lo = mid + 1` and `hi = mid`. We can eliminate the branch by computing both values and selecting with a conditional move (`cmov`):

```rust
fn lower_bound_branchless(a: *const i32, n: usize, key: i32) -> usize {
    let mut base: *const i32 = a;
    let mut n = n;
    while n > 1 {
        let half = n / 2;
        let mid = unsafe { base.add(half) };
        base = if key > unsafe { *mid } { mid } else { base };
        n -= half;
    }
    if key > unsafe { *base } {
        (unsafe { base.offset_from(a) } as usize) + 1
    } else {
        (unsafe { base.offset_from(a) } as usize)
    }
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

```rust
use std::alloc::{alloc, Layout};

fn eytzinger(sorted: *const i32, n: usize) -> *mut i32 {
    let layout = Layout::array::<i32>(n).unwrap();
    let e = unsafe { alloc(layout) as *mut i32 };
    build(e, sorted, 0, n, 0);
    e
}

fn build(e: *mut i32, sorted: *const i32, lo: usize, hi: usize, idx: usize) -> usize {
    if lo >= hi {
        return idx;
    }
    let mid = lo + (hi - lo) / 2;
    unsafe { *e.add(idx) = *sorted.add(mid); }
    let next = build(e, sorted, lo, mid, 2 * idx + 1);
    build(e, sorted, mid + 1, hi, 2 * idx + 2)
}
```

Searching the Eytzinger layout:

```rust
fn lower_bound_eytzinger(e: *const i32, n: usize, key: i32) -> usize {
    let mut idx: usize = 0;
    let mut result: usize = n;
    while idx < n {
        unsafe {
            if key > *e.add(idx) {
                idx = 2 * idx + 2;
            } else {
                result = idx;
                idx = 2 * idx + 1;
            }
        }
    }
    result
}
```

Why does this help? The first few levels of the Eytzinger tree are contiguous in memory. Level 0 (root) is at index 0. Level 1 is at indices 1-2. Level 2 at indices 3-6. Level 3 at indices 7-14. The first ~3 iterations access indices 0, then 1-2, then 3-6 — all within the first cache line (64 bytes = 16 ints). The first few comparisons that were L3 misses in the standard layout are now L1 hits.

Performance: ~55ns per query (~110 cycles). ~3.3× over the standard, ~1.5× over the branchless standard layout. The Eytzinger layout combines with branchless search:

```rust
fn lower_bound_eytzinger_branchless(e: *const i32, n: usize, key: i32) -> usize {
    let mut idx: usize = 0;
    while idx < n {
        let left = 2 * idx + 1;
        let right = 2 * idx + 2;
        let go_right = key > unsafe { *e.add(idx) };
        idx = if go_right { right } else { left };
    }
    reconstruct(idx, n)
}
```

The branchless Eytzinger search: ~35ns per query (~70 cycles). ~5.1× over standard. The remaining bottleneck: the data dependency chain. Each iteration needs `idx` from the previous iteration. The CPU can't overlap iterations.

## Stage 3: Prefetching in Eytzinger

Even with the Eytzinger layout, the deeper levels of the tree (beyond the first few) still cause cache misses. We can issue prefetches for the children before we need them:

```rust
use std::intrinsics::prefetch_read_data;

fn lower_bound_eytzinger_prefetch(e: *const i32, n: usize, key: i32) -> usize {
    let mut idx: usize = 0;
    while idx < n {
        let left = 2 * idx + 1;
        let right = 2 * idx + 2;
        if left < n {
            unsafe {
                prefetch_read_data(e.add(2 * left + 1) as *const ::std::ffi::c_void, 3);
                prefetch_read_data(e.add(2 * left + 2) as *const ::std::ffi::c_void, 3);
            }
        }
        if right < n {
            unsafe {
                prefetch_read_data(e.add(2 * right + 1) as *const ::std::ffi::c_void, 3);
                prefetch_read_data(e.add(2 * right + 2) as *const ::std::ffi::c_void, 3);
            }
        }
        let go_right = key > unsafe { *e.add(idx) };
        idx = if go_right { right } else { left };
    }
    reconstruct(idx, n)
}
```

The `__builtin_prefetch` instructions are hints to the hardware — they don't stall the pipeline. While the current comparison resolves, the prefetcher brings the grandchildren into L1. By the time we reach them, they're ready.

Performance: ~28ns per query (~56 cycles). ~6.4× over standard. Prefetching overlaps memory latency with computation. The deeper the tree, the bigger the win (more levels can be prefetched).

## Stage 4: Removing the Last Branch (Predicated Search)

For the final iterations of the search (when the remaining range fits in a cache line), we can switch to a fully predicated approach: load the entire cache line, compare all elements in parallel with SIMD, and extract the result with bit operations:

```rust
use std::arch::x86_64::*;

fn lower_bound_final(e: *const i32, n: usize, key: i32) -> usize {
    let mut idx: usize = 0;
    while idx * 2 + 16 < n {
        let left = 2 * idx + 1;
        let right = 2 * idx + 2;
        let go_right = key > unsafe { *e.add(idx) };
        idx = if go_right { right } else { left };
    }

    let subtree_size = find_subtree_size(idx, n);

    unsafe {
        let v = _mm512_loadu_si512(e.add(idx) as *const __m512i);
        let vkey = _mm512_set1_epi32(key);
        let mask = _mm512_cmple_epi32_mask(v, vkey);
        let count = (mask as u16).count_ones() as usize - 1;
        reconstruct(idx, n, count)
    }
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
