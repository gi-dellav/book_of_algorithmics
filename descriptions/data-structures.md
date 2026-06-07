# Chapter: Data Structures Case Studies (`data-structures/`)

## Overview

Weight 12 in the book. This chapter applies the hardware knowledge from earlier chapters to the design and optimization of data structures. It covers binary search (with the Eytzinger layout), static B-trees (S-tree), segment trees (including Fenwick trees and wide segment trees), B⁻ trees (dynamic B+ tree variant), and has stubs for hash tables, bitmaps, and probabilistic filters. The `_index.md` notes that data structures have more dimensions of optimization than algorithms (throughput, latency, memory usage, multiple query types).

## Files and Content

| File | Status | Size | Description |
|------|--------|------|-------------|
| `_index.md` | Complete | 766 B | Introduction framing data structure optimization as multi-dimensional |
| `b-tree.md` | Complete | 21.9 KB | Dynamic B⁻ tree (B+ tree variant) with SIMD-accelerated search. Covers memory layout, SIMD rank computation (`_mm256_cmpgt_epi32`, `_mm256_movemask_ps`, popcount), insertion with mask-store trick, node splitting. Benchmarked against `std::multiset` (7-18x faster lookups) and `absl::btree` (3-8x faster). |
| `binary-search.md` | Complete | 33.6 KB | Binary search evolution: standard → branchless (cmov) → Eytzinger layout → prefetching in Eytzinger → removing the last branch via predication. Appendix on expected comparison count. The branchless version is ~4x faster than `std::lower_bound` on small arrays. |
| `bitset.md` | Draft/Stub | 46 B | Empty placeholder for "Bitmaps" |
| `filters.md` | Draft | 249 B | Stub with a single insight about Bloom filters being "inverse" of caches |
| `hash-tables.md` | Draft | 1.3 KB | Brief notes on open addressing vs. chaining with a simple cyclic-array implementation. Notes 2-3x speed difference but doesn't develop further. |
| `s-tree.md` | Complete | 33.9 KB | Static implicit B-tree (S-tree) with B=16 fitting in a cache line. SIMD search, hugepages, compile-time height optimization, B+ tree variant (S+ tree), and a dynamic variant with child pointers. ~15x over `std::lower_bound`. |
| `segment-trees.md` | Complete | 42.5 KB | The most comprehensive article. Evolution: pointer-based → implicit/Eytzinger → bottom-up iterative → branchless bottom-up → Fenwick tree (BIT) → wide (B-ary) segment trees with SIMD. Up to 200x speedup over pointer-based for prefix sum queries. Covers lazy propagation and non-reversible monoids. |

## Image Assets

51 entries in `img/` (45 images + `src/` subdirectory with 8 files). All graphs and diagrams support the performance analysis.

## Strengths

1. **`segment-trees.md` is the chapter's masterpiece**: The 42.5 KB article traces a single data structure through 6 progressively optimized implementations, each motivated by a specific hardware insight. The Fenwick tree discussion (including the cache associativity fix via "holes") and the wide segment tree SIMD implementation are outstanding.
2. **`binary-search.md` and `s-tree.md` form a perfect pair**: The former optimizes binary search within a single cache line; the latter extends to cache-efficient B-trees. Together they cover the full spectrum from scalar to SIMD to cache-oblivious.
3. **`b-tree.md` handles the dynamic case**: While S-trees are static, B⁻ trees show how to add insertions/deletions while retaining most SIMD optimizations. The mask-store trick for shifting keys is clever.
4. **Hardware-grounded throughout**: Every optimization is justified by a specific hardware property — cache lines, SIMD width, branch misprediction cost, page size, decode width.
5. **Strong benchmarking methodology**: All articles compare against standard library implementations and report concrete speedups.
6. **The Fenwick tree cache fix**: The observation that powers-of-two in Fenwick tree indices cause associativity conflicts, and the solution of inserting "holes," is a beautiful application of the associativity chapter.

## Areas for Improvement

1. **`hash-tables.md` is barely started**: A hash table article at weight 8 deserves a full treatment comparable to the segment tree or binary search articles. Open addressing, Robin Hood hashing, SIMD-accelerated probing, and Swiss tables deserve coverage.
2. **`bitset.md` is empty**: Bitsets/bitmaps are widely used (Roaring Bitmaps, bitset iteration, rank/select structures) and should be covered.
3. **`filters.md` is a single insight**: Bloom filters, cuckoo filters, and quotient filters deserve a real article with benchmarks.
4. **No coverage of trees**: AVL/red-black/B-trees in the classical sense. The chapter jumps directly to cache-optimized variants without explaining the classical versions they improve upon.
5. **No persistent/immutable data structures**: Functional/persistent data structures (e.g., persistent segment trees, RRB vectors) are increasingly relevant in multi-core and functional programming contexts.
6. **Limited discussion of memory usage**: The `_index.md` mentions memory usage as an optimization dimension, but few articles quantify it. Space overhead of Eytzinger vs. standard layout, B⁻ tree node utilization, etc. are mentioned only in passing.

## Recommendations

1. **Write the hash table article**: This is the highest-priority addition. Cover: open addressing with linear/quadratic probing, Robin Hood hashing, Swiss tables (abseil's design), SIMD-accelerated probing using `_mm_cmpeq_epi8`, and the memory-vs-speed tradeoffs of load factors.
2. **Write a bitmap article**: Cover bitset iteration (using `tzcnt`/`blsi`), rank/select structures, compressed bitmaps (Roaring), and SIMD popcount for dense operations.
3. **Expand `filters.md`**: Add Bloom filter implementation, optimal number of hash functions, blocked Bloom filters for cache efficiency, and comparison with cuckoo filters.
4. **Add a tree survey article**: A brief comparative article covering classical BST variants (AVL, red-black, treap, splay) with benchmarking, serving as a baseline for the cache-optimized variants that follow.
5. **Add memory-overhead analysis**: Create a table across all data structures comparing memory per element, cache line utilization, and internal fragmentation.
6. **Consider adding**: (a) A skip list article (cache-friendly vs. classic tradeoffs), (b) a "Design Patterns for Cache-Efficient Data Structures" summary article connecting the common techniques across all articles.
