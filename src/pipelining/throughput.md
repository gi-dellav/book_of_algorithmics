# Throughput Computing

There are two ways to make code faster: reduce the number of instructions, or execute more instructions per cycle. Throughput computing is about the latter — structuring your code to maximize the number of instructions the CPU can execute simultaneously.

## The Multiple Accumulators Trick

This is the single most practical technique in this chapter. It turns latency-bound loops into throughput-bound loops.

Consider summing an array:

```rust
let mut sum: f32 = 0.0;
for i in 0..n {
    sum += a[i];
}
```

Each iteration adds one element to `sum`. The next addition depends on the result of the previous one — a true data dependency (RAW hazard). The loop is **latency-bound**: it can execute at most one addition per 3 cycles (float add latency).

Unroll and use multiple accumulators:

```rust
let mut sum0 = 0.0f32;
let mut sum1 = 0.0f32;
let mut sum2 = 0.0f32;
let mut sum3 = 0.0f32;
for i in (0..n).step_by(4) {
    sum0 += a[i];
    sum1 += a[i+1];
    sum2 += a[i+2];
    sum3 += a[i+3];
}
let sum = (sum0 + sum1) + (sum2 + sum3);
```

Now the four additions per iteration are *independent* — the CPU can execute them in parallel on different execution units. The loop becomes **throughput-bound**: limited by how fast the CPU can start new additions, not how fast each one completes.

How many accumulators do you need? The formula:

```
accumulators_needed = latency × throughput_per_cycle
```

For float add on Zen 2: `3 × 2 = 6` accumulators needed to saturate both FMA pipes. With 6 accumulators, you issue 2 adds per cycle, and each accumulator gets an add every 3 cycles — exactly matching the latency. Fewer accumulators and the CPU has idle pipes; more and you waste registers without benefit (but extra accumulators don't hurt).

For float FMA (fused multiply-add) on Zen 2: `4 × 2 = 8` accumulators.

## Unrolling vs. Accumulators

Loop unrolling and multiple accumulators serve different purposes:

- **Unrolling** reduces loop overhead (the decrement, compare, branch). The CPU can only execute one branch per cycle, so unrolling moves the overhead from "every iteration" to "every N iterations."
- **Multiple accumulators** break dependency chains, enabling more ILP. Without them, unrolling alone doesn't help a latency-bound loop.

The two techniques synergize: unroll enough to feed all accumulators each iteration without excessive loop overhead.

```rust
// Unrolled 8×, 8 accumulators — saturates Zen 2 FMA pipes:
let mut s0 = 0.0f32; let mut s1 = 0.0f32; let mut s2 = 0.0f32; let mut s3 = 0.0f32;
let mut s4 = 0.0f32; let mut s5 = 0.0f32; let mut s6 = 0.0f32; let mut s7 = 0.0f32;
for i in (0..n).step_by(8) {
    s0 += a[i];   s1 += a[i+1];
    s2 += a[i+2]; s3 += a[i+3];
    s4 += a[i+4]; s5 += a[i+5];
    s6 += a[i+6]; s7 += a[i+7];
}
```

But this is for scalar code. SIMD does even better: one AVX2 instruction processes 8 floats at once. With SIMD + 2 accumulators, you hit the same throughput with much less unrolling. The SIMD chapter covers this.

## The General Pattern

For any reduction loop (sum, product, min, max, bitwise AND):

1. Identify the latency L of the reduction operation.
2. Identify the throughput T (instructions per cycle).
3. Use at least `L × T` accumulators.
4. Unroll to feed those accumulators each iteration and amortize loop overhead.

```rust
// Generic reduction template
const UNROLL: usize = 8;
const ACCUM: usize = 8;
let mut acc = [0.0f32; ACCUM];
for i in (0..n - UNROLL + 1).step_by(UNROLL) {
    for j in 0..UNROLL {
        acc[j % ACCUM] += a[i + j];
    }
}
// Handle remainder
// Combine accumulators
let mut result = 0.0f32;
for j in 0..ACCUM {
    result += acc[j];
}
```

## The Critical Path as a DAG

Every computation has a dataflow graph — a directed acyclic graph (DAG) where nodes are operations and edges are data dependencies. The **critical path** is the longest path through this DAG, measured in cycles (using instruction latencies as edge weights).

The DAG for `sum += a[i]` in a loop:
```
load a[i]   (4 cycles, independent per iteration)
    ↓
    +       (3 cycles, DEPENDS on previous +)
    ↓
   sum      (feeds into next iteration's +)
```
The critical path is 3 cycles per iteration (the load latency is hidden by the OoO engine because loads from successive iterations are independent).

With 4 accumulators:
```
load a[i]     load a[i+1]    load a[i+2]    load a[i+3]
    ↓             ↓               ↓               ↓
    +             +               +               +        (all independent!)
    ↓             ↓               ↓               ↓
  sum0          sum1            sum2           sum3
```
Critical path is still 3 cycles, but now each cycle can dispatch 4 independent operations — 4× throughput.

## Quantifying Throughput Limits

To calculate the upper throughput limit of a loop:

1. **Front-end bound**: Can the fetch/decode keep up? Zen 2 decodes 4 x86 instructions/cycle. If your loop body averages 2 instructions per iteration, you can issue 2 iterations per cycle (front-end is not the bottleneck).

2. **Back-end bound by ports**: Sum the µop counts per execution port. If port 0 handles 2 µops/iteration and your loop runs at 0.25 iterations/cycle, port 0 is at 50% utilization — not the bottleneck. If some instruction uses a port that can only handle 0.25 µops/cycle (like division), that port is the bottleneck.

3. **Back-end bound by dependencies**: The critical path length sets a minimum cycle count. If the critical path is 30 cycles and you do 100 operations, the maximum throughput is 100/30 ≈ 3.3 operations per cycle (if they can be parallelized).

4. **Memory bound**: If you touch more data than fits in cache, the memory system's bandwidth becomes the bottleneck. Chapter 9 covers this in depth.

The interaction of these limits is captured by the **roofline model** (see `limits.md`). For most hot loops, the bottleneck is either the dependency chain (latency) or memory bandwidth — the front-end and port pressure are rarely the limit for well-optimized scalar code.
