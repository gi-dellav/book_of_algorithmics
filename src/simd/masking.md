# SIMD Masking and Blending

Branches are expensive. In scalar code, we use `cmov` and arithmetic tricks to eliminate them. In SIMD, we use masks and blends — computing both sides of a conditional and selecting elements without branching.

## Comparison Intrinsics

```rust
let a = unsafe { _mm256_loadu_ps(x) };
let b = unsafe { _mm256_loadu_ps(y) };
let mask = unsafe { _mm256_cmp_ps::<_CMP_GT_OQ>(a, b) };  // a > b

// mask[i] = 0xFFFFFFFF if a[i] > b[i], 0x00000000 otherwise
```

The result is a vector of all-ones (true) or all-zeros (false) — the SIMD equivalent of a condition mask.

## Masking with AND/ANDNOT

```rust
// Branchy scalar:
for i in 0..n {
    b[i] = if a[i] > 0.0 { a[i].sqrt() } else { 0.0 };
}

// Branchless SIMD:
let va = unsafe { _mm256_loadu_ps(a.add(i)) };
let zero = unsafe { _mm256_setzero_ps() };
let mask = unsafe { _mm256_cmp_ps::<_CMP_GT_OQ>(va, zero) };
let sqrt_result = unsafe { _mm256_sqrt_ps(va) };
let result = unsafe { _mm256_and_ps(sqrt_result, mask) };  // Zero where condition fails
unsafe { _mm256_storeu_ps(b.add(i), result) };
```

The `sqrt` is computed for all elements, including those we'll mask out. For sqrt (latency 13, throughput 6), the extra work is minimal. For expensive functions, compute only when the mask is often true.

## Blending

`blendv` selects elements from one of two vectors based on a mask:

```rust
let result = unsafe { _mm256_blendv_ps(false_val, true_val, mask) };
// result[i] = mask[i] ? true_val[i] : false_val[i]
```

For the ternary pattern `(condition) ? a : b`, use blend:
```rust
let result = unsafe { _mm256_blendv_ps(b, a, mask) };  // Equivalent to: mask ? a : b
```

## Masked Loads and Stores (AVX-512)

AVX-512 introduces mask registers (k0–k7) and masked memory operations:

```rust
let mask = unsafe { _mm512_cmp_ps_mask::<_CMP_GT_OQ>(a, b) };  // Mask in k-register
let result = unsafe { _mm512_maskz_loadu_ps(mask, ptr) };       // Load where mask is 1, zero elsewhere
unsafe { _mm512_mask_storeu_ps(ptr, mask, result) };            // Store where mask is 1
```

Masked loads/stores avoid the need for explicit blend — the hardware skips the load/store for masked-out lanes. This is both faster and avoids touching memory that the program shouldn't access (e.g., elements past the end of an array).

On AVX2 (no k-registers), you can simulate masked stores with `maskload`/`maskstore`:
```rust
let mask_bits = ...;  // All-ones or all-zeros per lane
unsafe { _mm256_maskstore_ps(ptr, mask_bits, value) };
```

`maskstore` is implemented as a series of scalar stores internally — it's slow. Prefer explicit blending + regular stores when possible.

## Case Study: SIMD `find`

Find the index of the first element equal to a target value:

```rust
// Scalar:
fn find_scalar(a: &[i32], target: i32) -> Option<usize> {
    a.iter().position(|&x| x == target)
}

// SIMD (AVX2):
unsafe fn find_avx2(a: *const i32, n: usize, target: i32) -> isize {
    let vtarget = _mm256_set1_epi32(target);
    let mut i = 0;
    while i + 7 < n {
        let va = _mm256_loadu_si256(a.add(i) as *const __m256i);
        let cmp = _mm256_cmpeq_epi32(va, vtarget);
        let mask = _mm256_movemask_ps(_mm256_castsi256_ps(cmp));
        if mask != 0 {
            // Found! Trailing-zero count gives the exact position
            return i as isize + mask.trailing_zeros() as isize;
        }
        i += 8;
    }
    -1
}
```

`_mm256_movemask_ps` extracts the sign bit of each 32-bit lane into an 8-bit integer. After `_mm256_cmpeq_epi32` (which sets lanes to all-ones for matches), the sign bits form a bitmask of matching elements. `__builtin_ctz` (count trailing zeros) finds the first match.

This is ~4× faster than scalar for large arrays. The case study in `algorithms/argmin.md` pushes this further: 4 blocks of AVX2, hitting the decode width limit at ~43 GFLOPS.

## SIMD `count` with Multiple Accumulators

Count elements matching a condition:

```rust
unsafe fn count_avx2(a: *const i32, n: usize, threshold: i32) -> i32 {
    let vthreshold = _mm256_set1_epi32(threshold);
    let mut vcount0 = _mm256_setzero_si256();
    let mut vcount1 = _mm256_setzero_si256();
    
    let mut i = 0;
    while i + 15 < n {
        let v0 = _mm256_loadu_si256(a.add(i) as *const __m256i);
        let v1 = _mm256_loadu_si256(a.add(i + 8) as *const __m256i);
        let mask0 = _mm256_cmpgt_epi32(v0, vthreshold);
        let mask1 = _mm256_cmpgt_epi32(v1, vthreshold);
        // mask is all-ones (=-1), so subtraction ADDS 1
        vcount0 = _mm256_sub_epi32(vcount0, mask0);
        vcount1 = _mm256_sub_epi32(vcount1, mask1);
        i += 16;
    }
    // Horizontal sum of both accumulators...
    // Then horizontal sum the two accumulator results
}
```

Two independent accumulator chains → ILP doubled. Saturating the execution ports requires enough accumulators — the formula from `pipelining/throughput.md` applies to SIMD too.
