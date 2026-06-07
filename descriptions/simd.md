# Chapter: SIMD Parallelism (`simd/`)

## Overview

Weight 10 in the book. This chapter introduces SIMD (Single Instruction, Multiple Data) programming on x86. It starts with auto-vectorization (getting SIMD "for free"), then dives into intrinsics, data movement, masking/blending, reductions, and in-register shuffles. The `_index.md` hooks the reader with a 2x speedup from a one-line `#pragma GCC target("avx2")`.

## Files and Content

| File | Status | Size | Description |
|------|--------|------|-------------|
| `_index.md` | Complete | 3.0 KB | Motivating example: array sum 2x faster with AVX2 pragma. Overview of SSE/AVX/AVX-512. Power/downclocking caveat. |
| `auto-vectorization.md` | Complete | 5.7 KB | Auto-vectorization: what compilers can do, common blockers (unknown size, aliasing, alignment), `-fopt-info-vec`, SPMD model in Julia/OpenMP/ISPC |
| `intrinsics.md` | Complete | 12.8 KB | Intrinsics deep dive: SIMD register history (SSE→AVX→AVX-512), vector types (`__m128`, `__m256`, `__m512`), naming convention, GCC vector extensions, third-party libraries (Highway, EVE, VCL) |
| `masking.md` | Complete | 11.9 KB | Masking and blending: comparison intrinsics, `blendv`, `maskload`/`maskstore`, SIMD `find` (4 optimization stages, 10x speedup), SIMD `count` with dual accumulators |
| `moving.md` | Complete | 10.0 KB | Data movement: aligned vs. unaligned loads, register aliasing, extract/insert cost (surprisingly slow!), broadcast, gather/scatter, mapping vectors to arrays |
| `reduction.md` | Complete | 4.6 KB | Reductions: horizontal operations (sum, min), multiple accumulators for ILP, `hadd` intrinsics, `_mm_minpos_epu16` |
| `shuffling.md` | Complete | 10.0 KB | In-register shuffles: `pshufb` for nibble-based popcount (Wojciech Muła technique), filter/compact algorithm using permutation LUT + `permutevar8x32`, AVX-512 `compress` mention |

## Image Assets

6 images: `filter.svg`, `gather-scatter.png`, `gather.svg`, `hsum.png`, `intel-extensions.webp`, `simd.png`. All directly support the articles.

## Strengths

1. **Practical, progressive structure**: Auto-vectorization (easy wins) → intrinsics (control) → masking (branch elimination) → shuffling (advanced). The reader can stop at any level.
2. **`masking.md` is a tour de force**: The SIMD `find` implementation going from basic (20 GFLOPS) → using `testz` → 2 blocks (34 GFLOPS) → 4 blocks (43 GFLOPS, hitting decode width limit) is a perfect demonstration of iterative SIMD optimization. Every stage is justified.
3. **`shuffling.md` shows the power of permutations**: The popcount via `pshufb` and the filter/compact algorithm are beautiful examples of how in-register data movement enables algorithms that seem inherently scalar.
4. **Honesty about SIMD costs**: `moving.md` explicitly warns that extract/insert operations are "surprisingly slow" and that gather "won't give you a speedup." This candor is rare in SIMD tutorials.
5. **Good ecosystem awareness**: Mentions Highway, EVE, VCL, xsimd, GCC vector extensions, ISPC, and OpenMP — showing readers there are alternatives to raw intrinsics.
6. **The AVX-512 downclocking note**: The `_index.md` mention of frequency scaling when using 512-bit instructions is an important practical detail often omitted.

## Areas for Improvement

1. **No AVX-512 deep dive**: AVX-512 is mentioned throughout but never given a dedicated article. Key features like mask registers, `compress`/`expand`, `vpternlogd`, and `vpopcnt` are only briefly referenced.
2. **No NEON/Arm SIMD coverage**: The chapter is exclusively x86. Given the growing importance of Arm (Apple M1/M2, AWS Graviton, mobile), even a brief comparison would add value.
3. **Missing: SIMD-friendly algorithm design patterns**: A summary article covering the common patterns (vectorized lookup tables, prefix-sum-on-registers, conflict-detection) would help readers generalize from the specific examples.
4. **`auto-vectorization.md` could go deeper**: The article mentions `#pragma GCC ivdep` and `__restrict__` but doesn't show a systematic debugging process for a loop that refuses to vectorize. A before/after case study with `-fopt-info-vec-missed` output would be instructive.
5. **No SIMD sorting**: Sorting networks (bitonic, Batcher's odd-even) are natural SIMD applications. An article on SIMD sorting would complement the existing `find`, `count`, `filter`, and `reduce` operations.
6. **The shuffle table in `shuffling.md`**: The permutation lookup table `permutation[256][8]` maps 8-bit masks to permutations. The construction of this table is not shown, only referenced.

## Recommendations

1. **Add an AVX-512 article**: Cover masked operations, `compress`/`expand`, ternary logic (`vpternlogd`), and the interaction with downclocking. Compare AVX2 vs. AVX-512 performance for the algorithms already presented.
2. **Add Arm NEON appendix**: Even a short comparison table (register widths, equivalent intrinsics, key differences like no `movemask`) would broaden the chapter's applicability.
3. **Add SIMD design patterns**: Summarize the recurring techniques: (a) vectorized lookup tables via `pshufb`, (b) building masks from comparisons, (c) horizontal reduction patterns, (d) interleaved multi-accumulator loops, (e) prefix-sum-on-register.
4. **Expand auto-vectorization debugging**: Add a case study where a loop fails to vectorize, show the `-fopt-info-vec-missed` output, and systematically fix each issue (aliasing → `__restrict__`, alignment → `alignas`, unknown trip count → `#pragma GCC ivdep`).
5. **Add SIMD sorting**: Implement a small sorting network (e.g., 8-element or 16-element) using `_mm256_min_epi32`/`_mm256_max_epi32` and compare to `std::sort` for small arrays.
6. **Show the permutation table construction**: In `shuffling.md`, include the code that generates the `permutation[256][8]` lookup table from mask → compressed indices.
