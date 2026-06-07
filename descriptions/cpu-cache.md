# Chapter: RAM & CPU Caches (`cpu-cache/`)

## Overview

Weight 9 in the book. The most extensive chapter with 12 articles. It experimentally characterizes the CPU cache hierarchy (L1, L2, L3, RAM) by running microbenchmarks that measure latency, bandwidth, associativity, cache line effects, prefetching, and memory-level parallelism. The `_index.md` acknowledges inspiration from Igor Ostrovsky's "Gallery of Processor Cache Effects" and Ulrich Drepper's "What Every Programmer Should Know About Memory."

## Files and Content

| File | Status | Size | Description |
|------|--------|------|-------------|
| `_index.md` | Complete | 5.2 KB | Experimental setup (Ryzen 7 4700U, Zen 2 specs), methodology, and acknowledgements |
| `alignment.md` | Complete | 8.6 KB | Aligned allocation (`alignas`, `std::aligned_alloc`), structure alignment/padding, optimizing member order, structure packing (`__attribute__((packed))`), bit fields |
| `aos-soa.md` | Complete | 6.3 KB | Array of Structures vs. Structure of Arrays, temporary storage contention, huge pages interaction, padded AoS, and RAM row buffer effects |
| `associativity.md` | Complete | 9.0 KB | Cache associativity explained via strided access experiment. Fully associative → direct-mapped → set-associative caches. Address composition (tag, index, offset). Practical fix: avoid powers of two. |
| `bandwidth.md` | Published | 7.5 KB | Memory bandwidth measurement via incrementing loop. L1/L2/L3/RAM bandwidth cliffs. Directional access (read-only vs. write-only vs. read+write). Non-temporal stores (`_mm256_stream_si256`). |
| `cache-lines.md` | Complete | 2.5 KB | Cache lines as the unit of data transfer. Strided access demonstrating same cache line count = same time. Padding for latency benchmarks. |
| `latency.md` | Complete | 4.7 KB | Pointer chasing to measure pure latency. Theoretical latency formula, frequency scaling effects. |
| `mlp.md` | Complete | 2.6 KB | Memory-Level Parallelism: latency × bandwidth product, direct measurement with D parallel pointer-chasing streams. ~6 for L2, ~13-17 for larger memory. |
| `paging.md` | Complete | 6.3 KB | TLB effects via strided access. Two TLB levels. Huge pages (2M/1G), `madvise`, and their impact on performance. |
| `pointers.md` | Complete | 4.1 KB | Pointer alternatives: indices vs. actual pointers (2ns vs. 3ns L1), 64-bit vs. 32-bit pointer overhead, 24-bit bit fields as a space optimization |
| `prefetching.md` | Complete | 6.9 KB | Hardware prefetching (pattern detection), software prefetching (`__builtin_prefetch`, `_mm_prefetch`), LCG-based prefetch distance experiment, D-ahead prefetching |
| `sharing.md` | Complete | 4.1 KB | Multi-core cache sharing, NUMA, CPU affinity (`taskset`), parallel bandwidth scaling, L3 cache topology (`lstopo`) |

## Image Assets

39 images — the most of any chapter. Includes many SVG performance graphs (bandwidth, latency, associativity, prefetching, sharing), cache diagrams, and the `lstopo` screenshot. All images serve the articles they appear in.

## Strengths

1. **Experiment-driven pedagogy**: This is the chapter's defining strength. Every concept is demonstrated with a microbenchmark and a performance graph. Readers don't need to trust theory — they see the cliffs, spikes, and plateaus.
2. **`associativity.md` is a standout**: The "which loop is faster, stride 256 or stride 257?" puzzle and its 10x answer is a brilliant hook. The progression from fully-associative → direct-mapped → set-associative with diagrams makes a complex topic intuitive.
3. **`bandwidth.md` is thorough**: Covers frequency scaling, directional access, non-temporal stores, and the subtle reason non-temporal writes are faster than reads.
4. **`aos-soa.md` reveals deep hardware behavior**: The interaction between AoS/SoA, huge pages, L3 physical vs. virtual addressing, and RAM row buffers is incredibly detailed — the kind of knowledge that only comes from extensive experimentation.
5. **Comprehensive coverage**: From cache line basics through TLB, prefetching, and multi-core sharing, virtually every cache-related performance topic is addressed.
6. **Practical recommendations**: Every article ends with actionable advice: "don't use powers of two," "use non-temporal stores for write-only data," "group hot code together," "enable huge pages."

## Areas for Improvement

1. **No summary/cheat sheet**: With 12 dense articles, a summary table of cache parameters (size, latency, bandwidth per level) and optimization rules of thumb would be extremely valuable for reference.
2. **Steep entry barrier**: The chapter assumes familiarity with C++ intrinsics (`_mm256_*`) and low-level memory concepts. An "if you're new to this" preface would help.
3. **`latency.md` formula is intimidating**: The theoretical latency derivation with multiple fractions and subscripts might lose readers. A simpler "here's what matters" summary would help.
4. **Missing coverage**: (a) Cache write policies (write-through vs. write-back), (b) cache coherence protocols (MESI) — mentioned in passing but not explained, (c) instruction cache behavior is not benchmarked (only data cache).
5. **The `sharing.md` article is brief**: At 4.1 KB, it could go deeper into NUMA-aware allocation (`libnuma`), page migration, and the performance implications of cross-socket memory access.
6. **`pointers.md` assumes 32-bit mode**: The 32-bit compatibility mode benchmark is interesting but requires special setup. A note on practical relevance (e.g., using 32-bit indices in large hash tables, or the `x32` ABI) would help.

## Recommendations

1. **Add a summary page**: A table or diagram showing the memory hierarchy with measured latencies and bandwidths, plus the top 10 cache optimization rules (use SoA for scanning, AoS for lookups, avoid power-of-two strides, etc.).
2. **Add cache coherence content**: A brief article on false sharing (`std::hardware_destructive_interference_size`), MESI protocol basics, and why `volatile` is not sufficient for multi-threaded code.
3. **Expand `sharing.md`**: Add NUMA allocation, `move_pages`, and benchmarks showing the penalty for cross-socket access.
4. **Add instruction cache benchmarks**: Measure I-cache effects (function size vs. fetch bandwidth, branch density vs. decode limits).
5. **Soften the entry barrier**: Add a "Cache Primer" as the first article — a 1-page summary of what caches are, why they exist, and the key performance numbers without requiring any C++ knowledge.
6. **Benchmark on a second architecture**: All experiments are on Zen 2. Adding a few key graphs from an Intel platform (e.g., Alder Lake) would demonstrate which effects are universal and which are architecture-specific.
