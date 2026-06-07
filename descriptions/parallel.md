# Chapter: Parallel Computing (`parallel/`)

## Overview

Weight 100, marked `draft: true`, part "Parallel Computing." A scaffold chapter with 14 files across 4 subdirectories. The `_index.md` frames the chapter around Moore's law ending, the shift to multicore, and the promise of parallel hardware. Subdirectories cover concurrency models (processes, threads, fibers, event-driven), synchronization primitives, parallel algorithms (OpenMP), and GPU programming (CUDA/PyCUDA). Most articles are drafts, stubs, or rough notes.

## Files and Content

| File | Status | Size | Description |
|------|--------|------|-------------|
| `_index.md` | Draft | 2.0 KB | Introduction: Moore's law, shift from frequency to multicore, hyperthreading, multi-socket, GPUs |
| `runtimes.md` | Draft/Stub | 56 B | Placeholder for "Threading Runtimes" |
| `algorithms/_index.md` | Empty | 45 B | Placeholder for "Parallel Algorithms" |
| `algorithms/openmp.md` | Empty | 22 B | Placeholder for "OpenMP" |
| `concurrency/_index.md` | Draft | 478 B | Concurrency introduction (3 sentences) |
| `concurrency/event-driven.md` | Draft | 1.2 KB | Event-driven concurrency: JavaScript callbacks, Actor model (Akka/Scala example) |
| `concurrency/fibers.md` | Draft | 599 B | Fibers/goroutines: Go example, N:M scheduling |
| `concurrency/processes.md` | Draft | 1.0 KB | Fork-based multiprocessing: C fork() example, isolation properties |
| `concurrency/threads.md` | Draft | 3.3 KB | Threads: C++ merge sort with pthreads, Python threading with GIL limitation, processes as workaround |
| `gpu/_index.en.md` | Draft | 16.8 KB | GPU Programming: Jupyter notebook-style article covering PyCUDA basics, memory management, reduction, matrix multiplication, bitonic sort. Very rough. |
| `gpu/cuda.md` | Empty | 20 B | Placeholder for "CUDA" |
| `synchronization/_index.md` | Draft | 545 B | Synchronization: race condition on parallel `s += a[i]`. Abruptly ends mid-sentence. |
| `synchronization/mutex.md` | Empty | 42 B | Placeholder for "Mutual Exclusion" |

## Image Assets

1 image: `moores-law.jpg` (42.2 KB) — classic Moore's law transistor-count graph.

## Strengths

1. **Logical structure**: The four subdirectories (concurrency, synchronization, algorithms, GPU) cover the key dimensions of parallel computing.
2. **Language diversity**: Examples span C, C++, Python, Go, Scala, and CUDA — good for showing cross-language patterns.
3. **`gpu/_index.en.md` has ambitious scope**: Covers reduction, dynamic programming, matrix multiplication, and bitonic sort. The PyCUDA approach (Jupyter notebook style) is accessible.
4. **Good concurrency taxonomy**: Processes vs. threads vs. fibers vs. event-driven is a useful framework.

## Areas for Improvement

1. **Most articles are empty or barely started**: 8 of 14 files contain only frontmatter or a few sentences. This chapter has the lowest completion rate in the book.
2. **`gpu/_index.en.md` is a mess**: It mixes Python, C++, and CUDA code with broken output, `NameError` tracebacks embedded in the document, and informal stream-of-consciousness writing. The "work vs. latency" section and the stock price digression feel out of place.
3. **No practical synchronization content**: `synchronization/_index.md` ends mid-sentence. `mutex.md` is empty. There's no coverage of atomics, compare-and-swap, lock-free programming, or memory ordering.
4. **No parallel algorithm analysis**: The `algorithms/` subdirectory has only empty stubs. There's no discussion of work, span, Brent's theorem, Amdahl's law, or Gustafson's law.
5. **No connection to hardware**: The chapter doesn't connect parallel constructs to the hardware details covered in earlier chapters (e.g., cache coherence overhead, false sharing, NUMA effects on parallel performance).
6. **`threads.md` is mostly C++ merge sort code**: The article spends most of its space on a complete merge sort implementation rather than explaining thread concepts.

## Recommendations

1. **Write the synchronization articles**: This is the most critical gap. Cover: mutexes (std::mutex, futex), atomics (compare_exchange, fetch_add), memory ordering (acquire/release/relaxed), lock-free stacks/queues, RCU, and false sharing (`std::hardware_destructive_interference_size`).
2. **Rewrite `gpu/_index.en.md`**: Remove the broken notebook output, organize into clear sections, and add benchmarks. Focus on what makes GPU programming different from CPU: warp execution, coalesced memory access, shared memory, and occupancy.
3. **Add parallel algorithm foundations**: Cover work and span, Brent's theorem (W/p + S model), Amdahl's law, and practical examples (parallel reduction, parallel sort, parallel BFS).
4. **Add OpenMP content**: A practical article on `#pragma omp parallel for`, reduction clauses, scheduling strategies, and the performance implications of each.
5. **Connect to earlier chapters**: Show how SIMD (Chapter `simd/`) combines with threading, how cache coherence (Chapter `cpu-cache/`) affects parallel performance, and how pipelining (Chapter `pipelining/`) interacts with hyperthreading.
6. **Add a "Parallel Performance Pitfalls" article**: Cover false sharing, lock contention, load imbalance, oversubscription, and the NUMA allocation issues from `cpu-cache/sharing.md` in a parallel context.
