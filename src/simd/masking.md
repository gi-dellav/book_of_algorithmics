# SIMD Masking and Blending

Branches are expensive. In scalar code, we use `cmov` and arithmetic tricks to eliminate them. In SIMD, we use masks and blends — computing both sides of a conditional and selecting elements without branching.

## Comparison Intrinsics

```c
__m256 a = _mm256_loadu_ps(x);
__m256 b = _mm256_loadu_ps(y);
__m256 mask = _mm256_cmp_ps(a, b, _CMP_GT_OQ);  // a > b

// mask[i] = 0xFFFFFFFF if a[i] > b[i], 0x00000000 otherwise
```

The result is a vector of all-ones (true) or all-zeros (false) — the SIMD equivalent of a condition mask.

## Masking with AND/ANDNOT

```c
// Branchy scalar:
for (int i = 0; i < n; i++)
    if (a[i] > 0)
        b[i] = sqrtf(a[i]);
    else
        b[i] = 0;

// Branchless SIMD:
__m256 va = _mm256_loadu_ps(a + i);
__m256 zero = _mm256_setzero_ps();
__m256 mask = _mm256_cmp_ps(va, zero, _CMP_GT_OQ);
__m256 sqrt_result = _mm256_sqrt_ps(va);
__m256 result = _mm256_and_ps(sqrt_result, mask);  // Zero where condition fails
_mm256_storeu_ps(b + i, result);
```

The `sqrt` is computed for all elements, including those we'll mask out. For sqrt (latency 13, throughput 6), the extra work is minimal. For expensive functions, compute only when the mask is often true.

## Blending

`blendv` selects elements from one of two vectors based on a mask:

```c
__m256 result = _mm256_blendv_ps(false_val, true_val, mask);
// result[i] = mask[i] ? true_val[i] : false_val[i]
```

For the ternary pattern `(condition) ? a : b`, use blend:
```c
__m256 result = _mm256_blendv_ps(b, a, mask);  // Equivalent to: mask ? a : b
```

## Masked Loads and Stores (AVX-512)

AVX-512 introduces mask registers (k0–k7) and masked memory operations:

```c
__mmask8 mask = _mm512_cmp_ps_mask(a, b, _CMP_GT_OQ);  // Mask in k-register
__m512 result = _mm512_maskz_loadu_ps(mask, ptr);        // Load where mask is 1, zero elsewhere
_mm512_mask_storeu_ps(ptr, mask, result);                 // Store where mask is 1
```

Masked loads/stores avoid the need for explicit blend — the hardware skips the load/store for masked-out lanes. This is both faster and avoids touching memory that the program shouldn't access (e.g., elements past the end of an array).

On AVX2 (no k-registers), you can simulate masked stores with `maskload`/`maskstore`:
```c
__m256i mask_bits = ...;  // All-ones or all-zeros per lane
_mm256_maskstore_ps(ptr, mask_bits, value);
```

`maskstore` is implemented as a series of scalar stores internally — it's slow. Prefer explicit blending + regular stores when possible.

## Case Study: SIMD `find`

Find the index of the first element equal to a target value:

```c
// Scalar:
int find_scalar(int *a, int n, int target) {
    for (int i = 0; i < n; i++)
        if (a[i] == target) return i;
    return -1;
}

// SIMD (AVX2):
int find_avx2(int *a, int n, int target) {
    __m256i vtarget = _mm256_set1_epi32(target);
    for (int i = 0; i < n; i += 8) {
        __m256i va = _mm256_loadu_si256((__m256i*)(a + i));
        __m256i cmp = _mm256_cmpeq_epi32(va, vtarget);
        int mask = _mm256_movemask_ps(_mm256_castsi256_ps(cmp));
        if (mask) {
            // Found! Trailing-zero count gives the exact position
            return i + __builtin_ctz(mask);
        }
    }
    return -1;
}
```

`_mm256_movemask_ps` extracts the sign bit of each 32-bit lane into an 8-bit integer. After `_mm256_cmpeq_epi32` (which sets lanes to all-ones for matches), the sign bits form a bitmask of matching elements. `__builtin_ctz` (count trailing zeros) finds the first match.

This is ~4× faster than scalar for large arrays. The case study in `algorithms/argmin.md` pushes this further: 4 blocks of AVX2, hitting the decode width limit at ~43 GFLOPS.

## SIMD `count` with Multiple Accumulators

Count elements matching a condition:

```c
int count_avx2(int *a, int n, int threshold) {
    __m256i vthreshold = _mm256_set1_epi32(threshold);
    __m256i vcount0 = _mm256_setzero_si256();
    __m256i vcount1 = _mm256_setzero_si256();
    
    for (int i = 0; i < n; i += 16) {
        __m256i v0 = _mm256_loadu_si256((__m256i*)(a + i));
        __m256i v1 = _mm256_loadu_si256((__m256i*)(a + i + 8));
        __m256i mask0 = _mm256_cmpgt_epi32(v0, vthreshold);
        __m256i mask1 = _mm256_cmpgt_epi32(v1, vthreshold);
        // mask is all-ones (=-1), so subtraction ADDS 1
        vcount0 = _mm256_sub_epi32(vcount0, mask0);
        vcount1 = _mm256_sub_epi32(vcount1, mask1);
    }
    // Horizontal sum of both accumulators...
    // Then horizontal sum the two accumulator results
}
```

Two independent accumulator chains → ILP doubled. Saturating the execution ports requires enough accumulators — the formula from `pipelining/throughput.md` applies to SIMD too.
