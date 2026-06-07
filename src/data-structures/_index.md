# Multi-Dimensional Optimization

Data structures have more dimensions of optimization than algorithms. An algorithm has one job: compute a result. A data structure has at least two: answer queries AND support updates. Some add a third (memory usage) and a fourth (construction time). Optimizing for one dimension often hurts another — the art is knowing which tradeoffs matter for your workload.

## Why Data Structures Are Harder

With an algorithm like matrix multiplication, you optimize for one thing: GFLOPS. Cache blocking, SIMD, register tiling — all push GFLOPS up. There's no tradeoff.

With a data structure like a hash table, you care about:
- **Lookup latency** (how fast can I find a key?)
- **Insertion throughput** (how many keys/second can I add?)
- **Memory overhead** (how many extra bytes per key?)
- **Deletion support** (can I remove keys efficiently?)
- **Iterator stability** (can I scan all entries in order?)

Improving one hurts another. Robin Hood hashing reduces variance in lookup time but makes insertions more expensive. Swiss tables use SIMD for probing but have higher memory overhead than chaining. There is no "best" hash table — only the best for *your* workload.

## What This Chapter Covers

Each article takes a fundamental data structure and pushes it through progressive optimization stages. Every stage is motivated by a specific hardware insight from earlier chapters:

1. **Binary Search** — Standard → branchless (`cmov`) → Eytzinger layout → prefetching → predicated last step. The canonical example of turning unpredictable branches into data dependencies.
2. **Static B-Trees (S-Trees)** — Extending Eytzinger to B=16 nodes that fill a cache line. SIMD-accelerated search within a node. ~15× over `std::lower_bound`.
3. **Dynamic B-Trees (B⁻ Trees)** — Adding insertion and deletion to the S-tree design. SIMD rank computation, mask-store shifting, node splitting.
4. **Segment Trees** — Pointer-based → implicit → bottom-up iterative → branchless → Fenwick tree → wide B-ary with SIMD. Up to 200× speedup on prefix sum queries.
5. **Hash Tables** — Open addressing, Robin Hood hashing, SIMD probing, Swiss tables. The full modern design space.
6. **Bitmap Structures** — Bitset iteration with `tzcnt`, rank/select, compressed bitmaps (Roaring). SIMD popcount for dense operations.
7. **Bloom Filters** — The "inverse cache": a probabilistic set that trades accuracy for space. Blocked Bloom filters for cache efficiency, cuckoo filters.
8. **Tree Structures Survey** — Classical BST variants (AVL, red-black, treap, splay) compared as baselines for the cache-optimized variants.

## How to Read This Chapter

The articles build on each other. Binary search introduces the Eytzinger layout. S-trees extend Eytzinger to cache-line-sized nodes. B⁻ trees add mutability to S-trees. Segment trees apply the same blocking ideas to a different query structure. Read binary search and S-trees first — the techniques they introduce recur throughout.

Every article includes concrete benchmarks on Zen 2. The numbers are real, measured on an AMD Ryzen 3970X locked at 2.0 GHz. Your hardware will differ, but the ratios (speedup from each optimization) transfer.
