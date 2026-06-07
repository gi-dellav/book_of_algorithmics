# Levels of Optimization

Performance optimization is not a binary state — your code is not simply "slow" or "fast." There are levels. Knowing which level you're at, and what it takes to reach the next one, is half the battle.

## Level 0: It Works

The code produces correct output. No thought has been given to performance. Loops are written in whatever order seemed natural. Data structures are whichever ones were taught in the introductory course. The compiler's default flags are used, if any flags are specified at all.

Most code lives here. For most code, this is fine. The overwhelming majority of lines in any software system are not performance-critical. A text editor can afford to be Level 0. A game engine's inner rendering loop cannot.

**How to progress**: Turn on compiler optimizations (`-O2` or `-O3`). Use architecture-specific flags (`-march=native`). Measure.

## Level 1: Compiler-Optimized

The programmer enables the compiler's optimizer and uses basic profiling to find hotspots. Loops are written in a way the compiler can vectorize. Data is laid out contiguously. Release builds are used for measurement, not debug builds.

This level costs almost nothing to achieve — a few compiler flags and a modest amount of discipline. It often delivers a 2–5× speedup over Level 0.

**How to progress**: Learn what the compiler can and can't optimize. Use `restrict` to resolve aliasing ambiguity. Use `[[likely]]`/`[[unlikely]]` to guide branch prediction. Run PGO (profile-guided optimization).

## Level 2: Cache-Aware

The programmer understands the memory hierarchy and structures data accordingly. Arrays of structs become structs of arrays where the access pattern demands it. Hot data is packed to minimize cache line usage. Loops are tiled to fit working sets into L1/L2 cache. Pointer-based data structures are replaced with index-based ones where possible.

This is where asymptotic complexity stops being the whole story. A cache-aware O(n log n) algorithm can beat a cache-oblivious O(n) algorithm for realistically-sized inputs. The programmer needs to understand cache line size, associativity, and bandwidth at each level of the hierarchy.

**Typical speedup**: Another 2–10×, sometimes dramatically more.

**How to progress**: Learn to read hardware performance counters. Run Cachegrind. Experiment with memory layouts and measure the impact.

## Level 3: Architecture-Exploiting

The programmer understands the CPU's internal execution model: pipelining, superscalar dispatch, branch prediction, execution ports, µop fusion. Code is written to avoid pipeline hazards. Dependency chains are broken with multiple accumulators. Branches are eliminated or made highly predictable. Hot loops are unrolled to the sweet spot where decode bandwidth and execution port pressure are balanced.

This level requires reading the CPU's optimization manual and understanding its specific execution resources. The code becomes less portable — it's tuned for a specific microarchitecture.

**Typical speedup**: 1.5–3× over Level 2 for compute-bound code.

**How to progress**: Use llvm-mca to analyze instruction scheduling. Study Agner Fog's instruction tables. Learn which execution ports your critical instructions use.

## Level 4: SIMD-Exploiting

The programmer writes vectorized code, either through auto-vectorization (guiding the compiler) or by using SIMD intrinsics directly. Data is aligned, padded, and laid out to enable aligned vector loads. Branches are replaced with masking and blending. In-register shuffles perform data rearrangement that would be slow in scalar code.

SIMD is the single largest lever for compute-bound workloads — a single AVX-512 instruction can do 16× the work of a scalar instruction at the same latency.

**Typical speedup**: 2–8× over Level 3 for data-parallel code.

**How to progress**: Learn the intrinsics for your ISA (SSE4.2, AVX2, AVX-512). Study the masking, blending, and shuffling primitives. Understand gather/scatter tradeoffs.

## Level 5: Peak

The code approaches the theoretical limits of the hardware. Every execution port is saturated. Memory bandwidth is fully utilized. The working set fits in the appropriate cache level. Prefetching hides remaining memory latency. Multi-core scaling is near-linear.

At this level, further optimization requires changing the hardware or the algorithm itself. This is where BLAS libraries live. Most developers never need to reach this level, and the effort to get here is enormous — but for the few kernels that dominate execution time, it can be worth it.

**Role models**: Intel MKL, OpenBLAS, FFTW, and the SIMD-optimized kernels in video codecs and cryptography libraries.

## When to Optimize

Not all code should be optimized to Level 5. The cost in development time, portability, and maintainability grows exponentially with each level.

The Pareto principle applies: 20% of the code consumes 80% of the runtime. Profile first. Find the hotspots. Optimize those. Leave the rest at Level 0 or 1.

And before you optimize, ask:
1. **Is this code actually a bottleneck?** Measure, don't guess.
2. **Is the optimization worth the complexity?** A 10% speedup in a function that takes 1% of runtime is not worth obscure code.
3. **Can the compiler do it for me?** Modern compilers are very good. Check the assembly output before reaching for intrinsics.

The chapters that follow build from Level 0 upward. By the end of this book, you should be able to take a critical loop from Level 1 to Level 4, and understand what it would take to reach Level 5.
