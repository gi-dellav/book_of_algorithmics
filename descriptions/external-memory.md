# Chapter: External Memory (`external-memory/`)

## Overview

Weight 8 in the book. This chapter develops the theoretical foundations for memory-bound algorithms. It introduces the external memory model, surveys the memory hierarchy, covers eviction policies, cache-oblivious algorithms, external sorting, locality concepts, virtual memory, and list ranking. The `_index.md` frames the chapter around the question "how long does it take to add two numbers?" — illustrating the 10⁷× gap between L1 cache and HDD access.

## Files and Content

| File | Status | Size | Description |
|------|--------|------|-------------|
| `_index.md` | Complete | 3.9 KB | Motivational introduction: memory hierarchy variance and the need for an I/O model |
| `hierarchy.md` | Complete | 7.9 KB | Detailed survey of memory types: registers, L1/L2/L3, RAM, SSD, HDD, network storage. Volatile vs. non-volatile. Latency/bandwidth/cost table (2021). |
| `list-ranking.md` | Complete | 4.7 KB | List ranking via randomized independent set removal, merge step using external join. Application to Euler tour trees. |
| `locality.md` | Complete | 11.0 KB | Spatial and temporal locality. Depth-first vs. breadth-first merge sort, dynamic programming (knapsack locality), sparse table memory layout, AoS vs. SoA (row vs. columnar databases). |
| `management.md` | Draft/Stub | 56 B | Empty placeholder for "Memory Management" |
| `model.md` | Complete | 3.3 KB | External memory model formal definition: N dataset, M internal memory, B block size, IOPS cost metric, SCAN(N) = ⌈N/B⌉ |
| `oblivious.md` | Complete | 9.2 KB | Cache-oblivious matrix transposition and matrix multiplication (Strassen). Analysis showing O(N³/(B√M)) I/O complexity. |
| `policies.md` | Complete | 6.9 KB | Cache eviction policies: FIFO, LRU, LIFO, MRU, LFU, RR, Bélády's optimal (MIN). Sleator-Tarjan theorem (LRU ≤ 2·OPT). LRU implementation using doubly-linked list + hash table. |
| `sorting.md` | Published | 10.3 KB | External merge sort: from 2-way to k-way, tall cache assumption, practical implementation with C code for block-wise sort + heap-based k-way merge, hash join vs. sort-merge join. |
| `sublinear.md` | Draft/Stub | 70 B | "Sketching" — empty stub for sublinear algorithms |
| `virtual.md` | Complete | 4.9 KB | Virtual memory: paging, page tables, TLB, mmap for file I/O, swap files. Nice explanation of why virtual memory exists (isolation, fragmentation, non-RAM access). |

## Image Assets

8 images: `hierarchy.png` (memory hierarchy pyramid), `k-way.png` (k-way merge), `list-ranking.png` (diagram), `memory-vs-compute.png` (compute vs. memory growth over time), `mergesort.png`, `opt.png` (OPT caching), `sparse-table.png` (RMQ diagram), `virtual-memory.jpg` (virtual→physical mapping).

## Strengths

1. **Excellent theoretical grounding**: The chapter provides the formal external memory model and uses it consistently to analyze algorithms. The tall cache assumption and its implications are clearly explained.
2. **`locality.md` covers diverse case studies**: Merge sort (depth vs. breadth), dynamic programming (knapsack), sparse table (4 layout variants), and AoS/SoA — each illustrates a different aspect of locality.
3. **`oblivious.md` is elegant**: The cache-oblivious matrix transpose and multiply algorithms are beautiful examples of recursive divide-and-conquer automatically adapting to any cache size. The I/O complexity analysis is rigorous.
4. **`sorting.md` is practical**: The transition from theory to C code (block-wise sort + k-way merge with a heap) makes external sorting concrete. The join discussion connects to real database use cases.
5. **`policies.md` includes the Sleator-Tarjan theorem**: This important theoretical result (LRU is competitive with OPT) is well-explained and provides confidence that theoretical analyses using OPT translate to real hardware.

## Areas for Improvement

1. **`management.md` and `sublinear.md` are empty stubs**: Memory management (allocation strategies, fragmentation, garbage collection) and sublinear algorithms (sketching, sampling, streaming) deserve content or should be removed.
2. **No connection to `cpu-cache/` chapter**: The external memory model analysis could be applied to cache levels, but there's no cross-reference showing that the same I/O formulas work for L1/L2/L3. This is a missed opportunity to unify the two chapters.
3. **`virtual.md` is incomplete**: It covers paging and mmap but doesn't discuss page faults, thrashing, working set size, or the performance cost of TLB misses (which are covered in `cpu-cache/paging.md` instead). The duplication is confusing.
4. **Missing external memory data structures**: B-trees are covered in `data-structures/`, but external memory-specific structures like buffer trees, Bε-trees, or LSM trees are not mentioned here.
5. **`list-ranking.md` is dense**: The algorithm is correct but the explanation is compressed. More intermediate steps and a worked example would improve comprehension.
6. **HDD/SSD specifics could be expanded**: The `hierarchy.md` table gives numbers, but there's no discussion of SSD write amplification, TRIM, or why random reads vs. sequential reads differ by orders of magnitude on HDDs but not SSDs.

## Recommendations

1. **Connect chapters**: Add a "From Theory to Practice" section at the end that maps external memory model parameters (M, B) to real cache levels and references the `cpu-cache/` benchmarks.
2. **Write `management.md` or remove it**: If written, cover memory allocators (ptmalloc, jemalloc, tcmalloc), fragmentation, and the performance implications of allocation patterns. Otherwise, delete the stub.
3. **Write `sublinear.md` or remove it**: A treatment of Count-Min Sketch, HyperLogLog, and reservoir sampling would enrich the chapter. Otherwise, remove.
4. **Merge virtual memory content**: Move `cpu-cache/paging.md`'s TLB content here, or add a cross-reference. Avoid having two articles independently explain TLBs.
5. **Add a B-tree/LSM-tree article**: Even a brief treatment comparing B-tree, Bε-tree, and LSM-tree write amplification and read amplification in the external memory model would be valuable.
6. **Add practical SSD considerations**: A short article on SSD internals (pages, blocks, garbage collection, write amplification) and how to design algorithms to be SSD-friendly (log-structured writes, avoiding small random writes).
