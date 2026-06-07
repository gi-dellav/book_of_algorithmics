# Sorting

Sorting is the most fundamental algorithmic primitive — database queries, search indexes, and data preparation pipelines all depend on it. Despite decades of optimization, the gap between `std::sort` and a hardware-aware radix sort can be 10× for integer keys. This article surveys the modern sorting landscape.

## Quicksort Optimization

`std::sort` is typically an **introsort**: quicksort with median-of-three pivot selection, switching to heapsort when recursion depth exceeds O(log n) (to guarantee O(n log n) worst-case), and switching to insertion sort for small subarrays (n < 16).

Optimizations beyond `std::sort`:

### Branchless Partitioning

The standard partitioning step has a conditional branch for element comparison. A branchless version uses `cmov`:

```rust
unsafe fn partition_branchless(a: *mut i32, lo: usize, hi: usize) -> usize {
    let pivot = *a.add(lo);
    let mut i = lo.wrapping_sub(1) as isize;
    let mut j = hi.wrapping_add(1) as isize;
    loop {
        // Branchless: always advance i and j, swap based on predicate
        loop { i += 1; if *a.add(i as usize) >= pivot { break; } }
        loop { j -= 1; if *a.add(j as usize) <= pivot { break; } }
        if i >= j { return j as usize; }
        // swap(a[i], a[j])
        a.add(i as usize).swap(a.add(j as usize));
    }
}
```

The comparison `a[i] < pivot` still has a branch, but for random pivots it's unpredictable ~50% of the time — so `cmov` wins. The branchless version is ~1.5× faster on random data.

### BlockQuicksort (Edelkamp & Weiß, 2016)

Process partitioning in blocks of 256 elements: compare 256 elements against the pivot, store comparison results as bitmasks, then swap the mismatched elements using the bitmasks. This reduces branch mispredictions and improves cache utilization.

### Pattern-Defeating Quicksort (pdqsort, Orson Peters, 2015)

Combines quicksort, heapsort, and insertion sort with pattern detection:
- Detects already-sorted data (O(n) pass).
- Detects "mostly sorted" data (limited inversions).
- Falls back to heapsort on adversarial inputs.
- Used in Rust's standard library (`slice::sort_unstable`).

## Radix Sort

For integer keys, radix sort is often faster than comparison-based sorting because it's branchless and cache-friendly.

### LSD Radix Sort (Least Significant Digit)

Sort by the least significant byte, then the next byte, ..., up to the most significant byte. Each pass is a counting sort:

```rust
unsafe fn radix_sort_lsd(a: *mut u32, n: usize) {
    use std::alloc::{alloc, dealloc, Layout};

    let layout = Layout::array::<u32>(n).unwrap();
    let buf = alloc(layout) as *mut u32;

    for byte in 0..4 {
        // Count occurrences of each byte value
        let mut count = [0usize; 256];
        let shift = byte * 8;
        for i in 0..n {
            count[((*a.add(i) >> shift) & 0xFF) as usize] += 1;
        }

        // Prefix sum of counts → starting positions
        let mut pos = [0usize; 256];
        for b in 1..256 {
            pos[b] = pos[b - 1] + count[b - 1];
        }

        // Scatter: place each element in its bucket
        for i in 0..n {
            let bucket = ((*a.add(i) >> shift) & 0xFF) as usize;
            *buf.add(pos[bucket]) = *a.add(i);
            pos[bucket] += 1;
        }

        // Swap arrays
        std::ptr::swap(a, buf);
    }

    dealloc(buf, layout);
}
```

Each pass does 2n reads + 2n writes = 4n memory operations × 4 passes = 16n total. The counting pass is sequential; the scatter pass has random writes into 256 buckets. The buckets are small enough (for n ≤ 10^6) to be L1-resident, so the random writes are to L1 — cheap.

Performance: For n = 1M random 32-bit integers, LSD radix sort is ~3× faster than `std::sort` on Zen 2. The gap widens with n (comparison-based sort's O(n log n) vs. radix sort's O(n × bytes)).

### MSD Radix Sort (Most Significant Digit)

Sort by the most significant byte first, recurse on each bucket. Better cache locality (buckets shrink quickly). Used in string sorting (American flag sort).

### In-Place Radix Sort (American Flag Sort)

A variant that doesn't require a secondary buffer, using in-place permutation within each bucket. Saves memory but has worse constant factors.

## SIMD Sorting Networks

For small arrays (n ≤ 64), sorting networks produce branchless, fixed-sequence code. A sorting network for 8 elements using AVX2:

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

// Sort 8 floats in a SIMD register
unsafe fn simd_sort_8(v: __m256) -> __m256 {
    // Bitonic sorting network: 6 stages of min/max swaps
    // Stage 1
    let v = simd_minmax_swap(v, 0, 1);  // Compare-swap lanes 0-1
    let v = simd_minmax_swap(v, 2, 3);
    let v = simd_minmax_swap(v, 4, 5);
    let v = simd_minmax_swap(v, 6, 7);
    // ... stages 2-6 ...
    v
}
```

A full sorting network for 8 elements has 19 compare-swap operations (6 stages × varying swaps per stage). With SIMD min/max instructions (`_mm256_min_ps`, `_mm256_max_ps`) and permute instructions for lane swaps, 8 elements can be sorted in ~20 cycles — much faster than `std::sort` on 8 elements (which has function call overhead and branch mispredictions).

**Bitonic Merge Networks**: A bitonic sequence (first increasing, then decreasing) can be merged into sorted order with O(n log n) SIMD operations. For small n (up to 64), bitonic sort is the SIMD sorting method of choice.

## Hybrid Algorithms

State-of-the-art sorting (Intel's `xss::sort`, Google's `vqsort`) combines:

1. **Quicksort** for the recursion framework (cache-friendly, O(n log n) average case).
2. **Radix sort** for partitions that are small enough to bucket efficiently.
3. **SIMD sorting networks** for base-case partitions (≤64 elements).
4. **Branchless partitioning** throughout.

## Performance Summary (Zen 2, 1M random uint32)

| Algorithm | Time | Speedup vs. `std::sort` |
|-----------|------|--------------------------|
| `std::sort` (introsort) | ~45 ms | 1× |
| pdqsort (Rust) | ~35 ms | 1.3× |
| LSD Radix Sort | ~15 ms | 3× |
| SIMD hybrid (vqsort) | ~10 ms | 4.5× |

## Key Lessons

1. **Comparison-based sorting is bottlenecked by branch mispredictions.** The `a[i] < pivot` decision is unpredictable for random data. Branchless partitioning and SIMD sorting networks eliminate this bottleneck.
2. **Radix sort wins for integers because it's branchless AND cache-friendly.** The 256 buckets fit in L1, and the scatter/gather is to L1 — much cheaper than random memory access.
3. **SIMD sorting networks are the fastest for small n** (≤64). They're entirely branchless, register-resident, and exploit the CPU's full execution width.
4. **The best sort is a hybrid.** Quicksort for cache-oblivious recursion, radix sort for medium partitions, SIMD networks for small ones. Each component shines at a different scale.
