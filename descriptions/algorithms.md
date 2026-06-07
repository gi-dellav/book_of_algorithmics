# Chapter: Algorithms Case Studies (`algorithms/`)

## Overview

This chapter contains 9 articles (3 of which are drafts or empty stubs) presenting in-depth case studies of real-world algorithm optimization. It sits at weight 11 in the book, positioned as the culmination of earlier architecture/cache/pipelining chapters. The `_index.md` frames it as the "scientifically valuable part of the book" where the reader applies CPU knowledge to harder-than-sum-of-array problems.

## Files and Content

| File | Status | Size | Description |
|------|--------|------|-------------|
| `_index.md` | Complete | 1.0 KB | Brief introduction positioning these as applied case studies |
| `argmin.md` | Complete | 15.4 KB | Computing argmin with SIMD. Explores scalar baseline, vector-of-indices approach, branch-based vs. comparison-then-index approaches. ~15x faster than scalar. |
| `factorization.md` | Complete | 23.0 KB | Integer factorization of word-sized integers. Walks through trial division, lookup tables, wheel factorization, precomputed primes, Pollard's rho, Pollard-Brent, and modulo optimizations. ~3x faster than previous state-of-the-art for 60-bit semiprimes. |
| `gcd.md` | Complete | 10.8 KB | Binary GCD derivation, ~2x faster than C++ standard library `std::gcd`. Covers Euclid's algorithm, binary GCD, and implementation tricks attributed to Daniel Lemire and Ralph Corderoy. |
| `logistic.md` | Draft | 4.0 KB | Optimizing logistic regression inference for MNIST-like classification. Explores quantization (char/byte weights) and removing unnecessary softmax. |
| `matmul.md` | Complete | 27.3 KB | Matrix multiplication from naive triple-loop to BLAS-level performance in <40 lines of C. Covers transposition, vectorization, memory efficiency, register reuse, kernel design, blocking, and generalizations. ~50x speedup from baseline. |
| `prefix.md` | Complete | 12.7 KB | Prefix sum with SIMD. Derives ~2.5x faster algorithm using vectorization, blocking, and continuous loads. |
| `reading-integers.md` | Draft | 1.3 KB | Brief notes on a claimed ~35x faster integer parsing algorithm. References SWAR techniques and AVX-512. |
| `sorting.md` | Stub | 45 B | Empty placeholder (only frontmatter with `draft: true`). |

## Image Assets

21 images in `algorithms/img/` support the complete articles with performance plots, diagrams, and code illustrations.

## Strengths

1. **Arc of complexity**: Articles progress from simpler (GCD) to extremely complex (factorization, matmul), teaching the reader to build on earlier techniques.
2. **Empirically grounded**: All articles include real benchmarks on specific hardware (Zen 2 @ 2GHz), with concrete performance numbers.
3. **Matmul as crown jewel**: The matmul article is particularly thorough — 7 stages from naive to BLAS-level, all under 40 lines of C, with clear explanations of blocking, register reuse, and kernel design.
4. **Acknowledges sources**: Properly credits prior work (Lemire, Corderoy, Pollard, Brent, etc.).
5. **Factorisation article is exceptional**: 8 distinct algorithm stages with clear progression, benchmarking each.

## Areas for Improvement

1. **Three articles are unfinished**: `logistic.md` (draft), `reading-integers.md` (draft notes), `sorting.md` (empty stub). These break the flow and promise unfulfilled content.
2. **Sorting is entirely missing**: Despite being arguably the most important algorithmic primitive, there is only an empty stub. This is a major gap.
3. **Reading-integers is underspecified**: The article claims "35x faster than scanf" but provides no actual implementation, just scattered notes with URLs and ideas.
4. **Logistic regression is shallow**: Only covers the trivial "remove softmax" and "use char quantization" optimizations. Doesn't discuss SIMD for the dot product, weight layout transformations, or more advanced quantization schemes.
5. **Missing algorithmic domains**: No graph algorithms, no string algorithms (beyond the integer-parsing stub), no computational geometry, no FFT/NTT.
6. **No cross-references between articles**: Articles don't reference each other's techniques even when applicable (e.g., matmul's blocking could reference prefix sum's blocking).

## Recommendations

1. **Write the sorting article**: This is the highest-priority gap. A sorting case study should cover quicksort optimization, radix sort vs. comparison sort tradeoffs, SIMD sorting networks, and possibly an intro to GPU sorting.
2. **Complete reading-integers**: Turn the scattered notes into a proper article. Include the SWAR parsing technique, the transpose-based approach, and benchmark against `scanf`, `strtol`, `from_chars`, etc.
3. **Fill in logistic regression**: Add SIMD dot-product, weight reordering for better cache behavior, mixed-precision (bfloat16), and batch inference.
4. **Add cross-references**: E.g., matmul's blocking technique → prefix sum's blocking; factorization's modulo tricks → GCD's binary GCD; argmin's branchless techniques → binary search's branchless variant.
5. **Consider adding**: (a) FFT and convolution algorithms, (b) graph shortest-path (Dijkstra optimization with radix heaps), (c) string searching (SIMD `strstr`, Boyer-Moore, Rabin-Karp with rolling hashes).
6. **Normalize benchmark methodology**: Some articles use Zen 2 @ 2GHz, some mention different hardware — standardize or at least note differences.
