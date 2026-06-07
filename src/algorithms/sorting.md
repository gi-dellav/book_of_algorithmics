# Sorting

Sorting is the most fundamental algorithmic primitive — database queries, search indexes, and data preparation pipelines all depend on it. Despite decades of optimization, the gap between `std::sort` and a hardware-aware radix sort can be 10× for integer keys. This article surveys the modern sorting landscape.

## Quicksort Optimization

`std::sort` is typically an **introsort**: quicksort with median-of-three pivot selection, switching to heapsort when recursion depth exceeds O(log n) (to guarantee O(n log n) worst-case), and switching to insertion sort for small subarrays (n < 16).

Optimizations beyond `std::sort`:

### Branchless Partitioning

The standard partitioning step has a conditional branch for element comparison. A branchless version uses `cmov`:

```c
int partition_branchless(int *a, int lo, int hi) {
    int pivot = a[lo];
    int i = lo - 1, j = hi + 1;
    while (1) {
        // Branchless: always advance i and j, swap based on predicate
        do { i++; } while (a[i] < pivot);
        do { j--; } while (a[j] > pivot);
        if (i >= j) return j;
        swap(a[i], a[j]);
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

```c
void radix_sort_lsd(uint32_t *a, int n) {
    uint32_t *buf = malloc(n * sizeof(uint32_t));
    
    for (int byte = 0; byte < 4; byte++) {
        // Count occurrences of each byte value
        int count[256] = {0};
        int shift = byte * 8;
        for (int i = 0; i < n; i++)
            count[(a[i] >> shift) & 0xFF]++;
        
        // Prefix sum of counts → starting positions
        int pos[256];
        pos[0] = 0;
        for (int b = 1; b < 256; b++)
            pos[b] = pos[b-1] + count[b-1];
        
        // Scatter: place each element in its bucket
        for (int i = 0; i < n; i++) {
            int bucket = (a[i] >> shift) & 0xFF;
            buf[pos[bucket]++] = a[i];
        }
        
        // Swap arrays
        uint32_t *temp = a; a = buf; buf = temp;
    }
    
    free(buf);
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

```c
// Sort 8 floats in a SIMD register
__m256 simd_sort_8(__m256 v) {
    // Bitonic sorting network: 6 stages of min/max swaps
    // Stage 1
    v = simd_minmax_swap(v, 0, 1);  // Compare-swap lanes 0-1
    v = simd_minmax_swap(v, 2, 3);
    v = simd_minmax_swap(v, 4, 5);
    v = simd_minmax_swap(v, 6, 7);
    // ... stages 2-6 ...
    return v;
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
