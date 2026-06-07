# The Three Tiers of Profiling

Imagine you're a physicist studying an unknown material. You have three tools:

1. **An optical microscope** — shows macroscopic structures. Limited resolution, but quick and easy.
2. **An electron microscope** — reveals nanoscale details. High resolution, but requires sample preparation and vacuum.
3. **A theoretical model** — predicts material properties from first principles. Powerful, but only as good as its assumptions.

Performance analysis works the same way:

1. **Instrumentation** (timers, counters in your code) — tells you *how long* something took. Fast, easy, low overhead. Resolution: milliseconds to microseconds.

2. **Statistical profiling** (`perf`, VTune) — samples the program counter and hardware counters thousands of times per second. Tells you *where* time was spent, *what* the CPU was doing (cache misses, branch mispredicts, instructions retired). Resolution: individual instructions, specific cache events.

3. **Simulation** (Cachegrind, llvm-mca) — models the CPU's behavior without running on real hardware. Tells you *what should happen* under idealized conditions. Resolution: per-instruction, per-cache-line, per-branch.

Each tier answers different questions:
- "Which function is slow?" → Instrumentation or statistical profiling.
- "Why is this function slow? Is it cache misses or branch mispredicts?" → Statistical profiling with hardware counters.
- "How close am I to the hardware's theoretical limit? Where is the bottleneck in this instruction sequence?" → Simulation.

This chapter covers all three tiers. By the end, you'll be able to profile any program, interpret the results, and act on them.

## The Cardinal Rule of Profiling

**Never optimize without measuring first.** Your intuition about where a program spends its time is wrong at least 50% of the time. This is not an insult — it's a law of software. Programs are too complex, compilers too clever, and hardware too counterintuitive for anyone to predict bottlenecks by reading source code.

Measure first. Profile. Find the 5% of code that consumes 80% of runtime. Optimize that 5%. Repeat.

Corollary: **Never "optimize" code you haven't profiled.** The change that makes code look "more efficient" often makes it slower (by confusing the compiler, increasing I-cache pressure, or adding dependencies the OoO engine can't resolve). If you haven't measured, you haven't optimized — you've just changed things.

## Profiling Decision Tree

1. **Is the program slow overall?** → `perf stat ./prog` (instruction count, IPC, cache miss rate).
2. **Which function is slow?** → `perf record ./prog; perf report` (sampling profiler).
3. **Which line of code within the slow function?** → `perf annotate` (instruction-level heat map) or Cachegrind.
4. **Is the loop memory-bound or compute-bound?** → `perf stat -e cache-misses,cache-references,instructions,cycles`.
5. **What is the critical path in this hot loop?** → `llvm-mca` or `uiCA`.
6. **Is the benchmark methodology sound?** → Review against `noise.md` and `benchmarking.md`.

Each tool has its place. An expert profiles from the top down: start broad, narrow down, then drill into the exact bottleneck.
