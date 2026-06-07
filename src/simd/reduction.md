# SIMD Reductions

A reduction combines all elements of a vector into a single value — sum, minimum, maximum, product. Horizontal operations (within a single vector) are unfortunately slow on x86, and efficient reductions require careful instruction selection.

## Horizontal Addition

To sum all 8 elements of an `__m256`:

```c
// Method 1: hadd (slow, microcoded)
__m256 hsum = _mm256_hadd_ps(v, v);          // [a+b, c+d, a+b, c+d, e+f, g+h, e+f, g+h]
hsum = _mm256_hadd_ps(hsum, hsum);           // [a+b+c+d, ...]
float sum = _mm256_cvtss_f32(hsum);          // Bottom element
```

`hadd` (horizontal add) is microcoded — it's implemented as a sequence of shuffles + vertical adds. It's slow (3–5 µops, latency ~6–8 cycles) and is often not the fastest approach.

```c
// Method 2: Manual shuffle + add (faster)
__m128 lo = _mm256_castps256_ps128(v);        // Low 128 bits
__m128 hi = _mm256_extractf128_ps(v, 1);       // High 128 bits
__m128 sum128 = _mm_add_ps(lo, hi);            // 4 partial sums
sum128 = _mm_hadd_ps(sum128, sum128);          // 2 partial sums
sum128 = _mm_hadd_ps(sum128, sum128);          // 1 final sum
float sum = _mm_cvtss_f32(sum128);
```

This uses `extractf128` + vertical add for the first step, then `hadd` for the remaining 4→2→1 reduction. Faster than 2× `hadd` on 256-bit vectors because `extractf128` is a single µop.

## General Reduction Pattern

For any associative reduction (sum, min, max, bitwise AND/OR/XOR):

1. **Vectorize the loop** with multiple accumulators (for ILP).
2. **After the loop**: horizontally reduce each accumulator.
3. **Combine accumulator results**.

```c
float sum_array_avx2(float *a, int n) {
    __m256 sum0 = _mm256_setzero_ps();
    __m256 sum1 = _mm256_setzero_ps();
    __m256 sum2 = _mm256_setzero_ps();
    __m256 sum3 = _mm256_setzero_ps();
    
    for (int i = 0; i < n; i += 32) {
        sum0 = _mm256_add_ps(sum0, _mm256_loadu_ps(a + i));
        sum1 = _mm256_add_ps(sum1, _mm256_loadu_ps(a + i + 8));
        sum2 = _mm256_add_ps(sum2, _mm256_loadu_ps(a + i + 16));
        sum3 = _mm256_add_ps(sum3, _mm256_loadu_ps(a + i + 24));
    }
    
    // Combine accumulators
    sum0 = _mm256_add_ps(sum0, sum1);
    sum2 = _mm256_add_ps(sum2, sum3);
    sum0 = _mm256_add_ps(sum0, sum2);
    
    // Horizontal reduction of sum0
    __m128 lo = _mm256_castps256_ps128(sum0);
    __m128 hi = _mm256_extractf128_ps(sum0, 1);
    __m128 sum = _mm_add_ps(lo, hi);
    sum = _mm_hadd_ps(sum, sum);
    sum = _mm_hadd_ps(sum, sum);
    return _mm_cvtss_f32(sum);
}
```

With 4 accumulators × 8 floats each = 32 elements processed per iteration, the loop is throughput-bound on the 2 load ports (2 loads/cycle, 4 loads/iteration → 2 cycles/iteration minimum).

## `_mm_minpos_epu16`

A special-purpose but fast instruction: finds the position and value of the minimum unsigned 16-bit integer in a 128-bit vector:

```c
__m128i v = _mm_setr_epi16(5, 3, 8, 1, 9, 2, 7, 4);
__m128i minpos = _mm_minpos_epu16(v);
// minpos: [1, 3] — minimum value 1 at position 3
```

Latency: 4 cycles, throughput: 1/cycle. Much faster than a manual min-reduction + `tzcnt`. Useful for argmin operations in image processing and data compression (find smallest delta). Unfortunately only 128-bit and only unsigned 16-bit — for 32-bit values, manual reduction is needed.

## AVX-512 Reductions

AVX-512 introduces dedicated reduction instructions (AVX-512F):

```c
float sum = _mm512_reduce_add_ps(v);   // Sum all 16 floats
float min = _mm512_reduce_min_ps(v);   // Minimum of 16 floats
```

These are implemented as microcode sequences but are well-optimized. Easier to write and read than manual shuffle sequences. Still, for maximum throughput, multiple accumulators + manual reduction is the pattern.

## When to Reduce Inside vs. Outside the Loop

For most loops, reduce **after** the loop (accumulate in vectors, reduce once at the end). Horizontal operations inside the loop body kill performance — they break vectorization into scalars.

Exception: when the reduction result is needed each iteration (e.g., finding the maximum for normalization in the next iteration). In these cases, the horizontal operation is unavoidable, but try to amortize it (process a large block, reduce, process next block).
