# Theoretical Limits

Every processor has hard physical limits. Before you spend time optimizing, compute these limits for your target hardware. If your code is already within a factor of 2 of the theoretical peak, further effort has diminishing returns. If it's at 10% of peak, there's room to improve — and the type of gap tells you what to fix.

## The Roofline Model

The roofline model (Williams, Waterman, Patterson, 2009) captures the interaction between compute and memory in a single graph.

Axes:
- **X-axis**: Operational Intensity (FLOPs per byte of memory traffic)
- **Y-axis**: Attainable Performance (GFLOPS)

Two ceilings form the "roofline":

1. **Peak Compute** (horizontal line): The processor's theoretical maximum FLOPS.
   - Zen 2, 1 core @ 2 GHz: 32 GFLOPS (single-precision FMA, two 256-bit units).

2. **Peak Memory Bandwidth** (diagonal line): Performance = Intensity × Bandwidth.
   - Zen 2, single-channel DDR4-3200: ~20 GB/s (theoretical), divided by core count.
   - For memory-bound code on one core: Performance ≤ Intensity × (25 GB/s / 4 cores) ≈ Intensity × 6.25 GB/s.

The actual performance is bounded by whichever ceiling is lower at the code's operational intensity.

**How to compute operational intensity**:
- Count total DRAM bytes transferred (not cache — only what goes to DRAM).
- Count total FLOPs executed.
- Intensity = FLOPs / DRAM_bytes.

**Interpreting the roofline**:
- If your code's intensity is below the ridge point (intersection of compute and bandwidth ceilings), it's **memory-bound**. Faster execution units won't help. You need to increase intensity (reuse data more, or read less data).
- If above the ridge point, it's **compute-bound**. Faster memory won't help. You need to reduce instruction count or increase ILP.

For Zen 2, the ridge point is roughly: 32 GFLOPS / (25 GB/s / 4 cores) ≈ 5 FLOPs/byte. If your algorithm does fewer than 5 operations per byte read from DRAM, you're memory-bound.

## Peak FLOPS Calculation

```
Peak GFLOPS = cores × frequency × (FP_units_per_core × vector_width / 32) × ops_per_FMA
```

- Zen 2: 1 core × 2.0 GHz × (2 FMA units × 256 bits / 32 bits) × 2 ops/FMA = 32 GFLOPS
- Zen 4: 1 core × 5.0 GHz × (2 FMA units × 256 bits / 32) × 2 = 160 GFLOPS (with AVX-512: 320 GFLOPS)

Double precision is half: 16 GFLOPS on Zen 2.

**Achievable fraction of peak**: Hand-tuned BLAS reaches ~90% for large matrices. A well-optimized simple loop may reach 50–70%. Any code below 10% has a clear bottleneck.

## Peak Memory Bandwidth

```
Peak BW = memory_clock × channels × bytes_per_channel × transfers_per_clock
```

- Single-channel DDR4-3200: 1.6 GHz × 1 × 8 bytes × 2 (DDR) = 25.6 GB/s
- Dual-channel: 51.2 GB/s
- L3 cache bandwidth on Zen 2: ~32 bytes/cycle × 2.0 GHz ≈ 64 GB/s (per CCX)
- L2 cache bandwidth: ~64 bytes/cycle ≈ 128 GB/s

The STREAM benchmark measures sustainable bandwidth. It's typically 70–85% of theoretical for reads, less for writes (write-allocate penalty).

## Sorting Lower Bound

For comparison-based sorting, the decision-tree lower bound is ⌈log₂(n!)⌉ ≈ n log₂ n − n/ln 2 + O(log n) comparisons. This is the absolute minimum number of comparisons needed to sort n elements — no algorithm, no matter how clever, can do better.

But comparison count is not the bottleneck! Cache efficiency, branch predictability, and SIMD utilization dominate. A radix sort does zero comparisons but often outperforms quicksort for integer keys, because:
- It's naturally cache-friendly (sequential access).
- It's branchless (counting into histogram buckets).
- It can be vectorized.

The theoretical lower bound for sorting in the external memory model is O((N/B) log_{M/B}(N/B)) I/Os. This bound is achievable (multiway merge sort), and it correctly predicts that comparison count is secondary to I/O.

## Decode Width Limit

The front-end can decode at most 4 x86 instructions per cycle (Zen 2). If each instruction is a SIMD operation on 8 floats, the decode limit translates to 32 floats/cycle = 64 GFLOPS (with FMA). The decode limit is well above the execution limit for Zen 2, but it can bottleneck code with very short instructions.

Intel's Gracemont (E-core) decodes only 2 instructions per cycle — the decode limit can be a real bottleneck there for SIMD-heavy loops.

## Retirement Width Limit

Zen 2 retires up to 8 µops/cycle. This is rarely the bottleneck, but it sets an upper bound: you cannot sustain more than 8 µops/cycle regardless of execution resources. A loop that issues exactly 4 µops/iteration at 2 iterations/cycle hits 8 µops/cycle retirement — at the limit.

## Other Physical Limits

**Power**: Modern CPUs have TDPs of 15–105 W. Sustained AVX-512 workloads can exceed this, causing the CPU to downclock (reduce frequency). The `_index.md` of the SIMD chapter discusses this.

**Thermals**: If the CPU temperature exceeds the thermal junction limit (typically 95–105°C), it throttles frequency. Sustained compute-bound workloads should be monitored for thermal throttling.

**Memory capacity**: If your working set exceeds RAM capacity, the OS starts paging to disk. Performance drops by 100–1000×. The external memory chapter covers this.

## Applying the Limits

Before optimizing, answer:

1. **What is this code's operational intensity?** → Is it memory-bound or compute-bound?
2. **What fraction of peak compute/bandwidth does it achieve?** → Is there room to improve?
3. **What is the critical path latency?** → If compute-bound, is the dependency chain the bottleneck?
4. **What is the port pressure?** → If compute-bound and not dependency-bound, which execution port is saturated?
5. **What is the memory bandwidth utilization?** → If memory-bound, are you using the full bandwidth, or is latency (pointer chasing) the real issue?

The answers tell you which optimization to pursue. If memory-bound: improve locality, compress data, use non-temporal stores. If compute-bound by dependency chains: break them with multiple accumulators. If compute-bound by port pressure: use different instructions that schedule to different ports. If compute-bound by decode width: reduce instruction count, use SIMD more aggressively.

Most code in the wild is memory-bound. The cache optimization chapters (8 and 9) contain the highest-leverage techniques in this book.
