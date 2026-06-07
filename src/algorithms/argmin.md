# Argmin with SIMD

Finding the index of the minimum element in an array — the "argmin" — is a fundamental building block. It's used in neural network softmax, shortest path algorithms, and database query processing. This article develops a SIMD-accelerated argmin that's ~15× faster than the scalar baseline.

## The Scalar Baseline

```c
int argmin_scalar(float *a, int n) {
    int best_idx = 0;
    float best_val = a[0];
    for (int i = 1; i < n; i++) {
        if (a[i] < best_val) {
            best_val = a[i];
            best_idx = i;
        }
    }
    return best_idx;
}
```

Performance: ~0.75 elements per cycle (Zen 2). The loop is dominated by the unpredictable branch (each new minimum triggers the branch, but minima become rarer as the loop progresses) and the 4-cycle load latency.

## Stage 1: SIMD Minimum with Index Tracking

The challenge: SIMD can find the minimum value easily (`_mm256_min_ps`), but we also need the *index* of that minimum. The approach: maintain a vector of indices alongside the values, and update both together:

```c
int argmin_avx2_vector_index(float *a, int n) {
    __m256 best_vals = _mm256_loadu_ps(a);
    __m256i best_indices = _mm256_setr_epi32(0, 1, 2, 3, 4, 5, 6, 7);
    __m256i indices_inc = _mm256_set1_epi32(8);
    __m256i current_indices = best_indices;
    
    for (int i = 8; i < n; i += 8) {
        current_indices = _mm256_add_epi32(current_indices, indices_inc);
        __m256 vals = _mm256_loadu_ps(a + i);
        __m256 mask = _mm256_cmp_ps(vals, best_vals, _CMP_LT_OQ);
        
        best_vals = _mm256_blendv_ps(best_vals, vals, mask);
        best_indices = _mm256_blendv_epi8(best_indices, current_indices,
                                           _mm256_castps_si256(mask));
    }
    
    // Horizontal reduction: find min among the 8 candidate values
    // and return the corresponding index
    ...
}
```

The comparison mask (all-ones where the new value is less than the current best) is used both to update the best values and the best indices via `blendv`. The `blendv` selects the new value where the mask is true, the old value where false — a branchless SIMD ternary.

Performance: ~2.5 elements/cycle (3.3× speedup).

## Stage 2: Branch-Based vs. Comparison-Then-Index

The comparison mask from `_mm256_cmp_ps` is an 8-bit integer after `movemask`. We can test it:

```c
int mask = _mm256_movemask_ps(mask_float);
if (mask) {
    // At least one element is a new minimum — find which one
    int local_idx = __builtin_ctz(mask);  // Position of first match
    ...
}
```

The `if (mask)` branch is highly predictable (new minima become rare as the loop progresses, so `mask` is usually 0). This branch-based approach avoids unnecessary `blendv` operations when there's no new minimum.

Performance: ~4.5 elements/cycle (6× speedup).

## Stage 3: Multiple Vectors per Iteration

Process 32 elements per iteration (4 × AVX2 vectors):

```c
for (int i = 0; i < n; i += 32) {
    __m256 v0 = _mm256_loadu_ps(a + i);
    __m256 v1 = _mm256_loadu_ps(a + i + 8);
    __m256 v2 = _mm256_loadu_ps(a + i + 16);
    __m256 v3 = _mm256_loadu_ps(a + i + 24);
    
    __m256 mask0 = _mm256_cmp_ps(v0, best, _CMP_LT_OQ);
    __m256 mask1 = _mm256_cmp_ps(v1, best, _CMP_LT_OQ);
    __m256 mask2 = _mm256_cmp_ps(v2, best, _CMP_LT_OQ);
    __m256 mask3 = _mm256_cmp_ps(v3, best, _CMP_LT_OQ);
    
    // Combine masks: if any bit is set, find the minimum among the 4 vectors
    int mask = (_mm256_movemask_ps(mask0)) |
               (_mm256_movemask_ps(mask1) << 8) |
               (_mm256_movemask_ps(mask2) << 16) |
               (_mm256_movemask_ps(mask3) << 24);
    
    if (mask) {
        // Find the position of the first set bit
        // Update best value and best index
    }
}
```

This reduces the loop overhead (one `cmp` + `jne` per 32 elements instead of per 8), and the `if (mask)` test remains predictable. The instruction-level parallelism of processing 4 independent vectors keeps all execution ports busy.

Performance: ~11 elements/cycle (14.7× speedup).

## The Final Implementation

Bringing it all together (~100 lines of C with intrinsics):

- 4 × AVX2 vectors per iteration (32 elements).
- Vector-of-indices approach for tracking positions.
- Branch-based mask check (predictable after the first few iterations).
- Manual horizontal reduction at the end.

**Performance: ~11 elements/cycle on Zen 2, ~15× faster than scalar.**

The implementation reaches 55% of peak L1 bandwidth, indicating we're memory-bandwidth-bound rather than compute-bound. Further optimizations would need to reduce memory traffic (e.g., computing argmin on the fly rather than storing full arrays).

## Extensions

- **Argmax**: Flip the comparison predicate.
- **Top-K**: Maintain a heap of the K smallest values+indices.
- **Argmin in a range**: Apply the mask only to elements in range.
- **Floating-point tie-breaking**: If two elements are equal, pick the one with the smaller index (already handled by the `_CMP_LT_OQ` predicate, which is strict less-than).
