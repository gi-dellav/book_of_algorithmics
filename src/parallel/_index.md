# Beyond the Single Core

For the first thirteen chapters of this book, we assumed a single CPU core doing one thing at a time. That was true for decades. Moore's law gave us faster transistors and deeper pipelines; Dennard scaling gave us more performance at the same power. Then, around 2005, both stopped. Transistors kept shrinking, but they no longer got faster. The solution: put more cores on the same die.

Today, even a mid-range laptop has 8 cores. A server has 64. A GPU has 10,000. Parallelism is no longer optional — it's the only path to higher performance. But parallelism introduces an entirely new dimension of complexity: concurrency, synchronization, memory ordering, and the combinatorial explosion of interleavings.

## Why Parallelism Is Hard

A sequential program has one execution path. A parallel program with n threads has an exponential number of possible interleavings. Two threads incrementing the same counter:

```c
// Thread A          // Thread B
int t = counter;      int t = counter;
t = t + 1;            t = t + 1;
counter = t;          counter = t;
```

If both threads read `counter` before either writes, the counter increases by 1 instead of 2. This is a **race condition**: the result depends on the non-deterministic interleaving of operations. Finding race conditions is hard; reproducing them is harder. They may only manifest under specific cache coherence states, with specific thread interleavings, on specific hardware.

The tools to tame this complexity:
- **Mutexes**: enforce mutual exclusion, turning parallel code back into sequential code (for critical sections).
- **Atomics**: hardware-guaranteed indivisible operations that enable lock-free data structures.
- **Memory ordering**: control how writes from one thread become visible to others.

## What This Chapter Covers

1. **Concurrency Models** — Processes, threads, fibers, and event-driven execution. The four fundamental ways to express concurrent work, from heavy isolation (processes) to lightweight cooperative multitasking (fibers).
2. **Synchronization** — Mutexes, atomics, and memory ordering. How to prevent race conditions without destroying performance.
3. **Parallel Algorithms** — Work and span analysis, Amdahl's law, parallel reduction, parallel sort, and OpenMP pragmas.
4. **GPU Programming** — The SIMT execution model, CUDA, warps, shared memory, and coalesced access. GPUs have 100× the memory bandwidth of CPUs but require radically different algorithms.

## Hardware Context

This chapter connects directly to the earlier hardware chapters:
- **Cache coherence** (Chapter cpu-cache): false sharing, MESI protocol, and why two threads writing to adjacent cache lines can destroy performance.
- **SIMD** (Chapter simd): GPUs are essentially SIMD machines with thousands of lanes. Understanding SIMD on CPU prepares you for GPU.
- **Pipelining** (Chapter pipelining): hyperthreading shares a core's execution units between two threads — understanding pipeline hazards helps predict when hyperthreading helps vs. hurts.

## Performance Philosophy

Parallel performance is not about making each core work faster — it's about keeping all cores busy with useful work. The metrics:
- **Speedup**: T₁ / Tₚ (time on 1 core / time on p cores). Ideal: p. Reality: usually 0.5p to 0.9p.
- **Efficiency**: speedup / p. How much of each core's potential are you using?
- **Scalability**: does speedup improve as you add more cores, or does it plateau?

The limits come from Amdahl's law: if a fraction s of your program is serial, maximum speedup is 1/s. If 10% is serial, you can never exceed 10× speedup regardless of how many cores you have. The corollary: optimize the serial portion first.
