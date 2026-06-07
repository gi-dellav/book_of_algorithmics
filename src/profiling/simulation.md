# Cache and Branch Simulation

When `perf stat` tells you "cache misses are high" but doesn't tell you *which line of code* caused them, you need a simulator. Cachegrind (part of Valgrind) simulates the cache hierarchy at the source-line level, annotating each line with the number of cache misses it caused.

## Valgrind and Cachegrind

Valgrind is a dynamic binary instrumentation framework. It runs your program on a synthetic CPU and can intercept memory accesses, branch decisions, and other events. Cachegrind uses this to simulate caches and branch predictors.

```bash
valgrind --tool=cachegrind ./program
```

This produces `cachegrind.out.<pid>`. The program runs ~20–50× slower than native (this is simulation, not profiling), but the counts are deterministic and reproducible.

## Viewing Results

```bash
cg_annotate cachegrind.out.<pid>
```

Output:
```
--------------------------------------------------------------------------------
-- Auto-annotated source: hot_loop.c
--------------------------------------------------------------------------------
Ir I1mr ILmr Dr D1mr DLmr Dw D1mw DLmw
-- ---- ---- ---- ---- ---- ---- ----
. . . . . . . . .  int sum_positive(int *a, int n) {
. . . . . . . . .      int total = 0;
. . . . . . . . .      for (int i = 0; i < n; i++) {
8,000 1 1 4,000 2,000 1,900 1,000 0 0          if (a[i] > 0)
8,000 0 0 4,000 1,000 500 2,000 0 0              total += a[i];
. . . . . . . . .      }
. . . . . . . . .      return total;
. . . . . . . . .  }
```

Column meanings:
- **Ir**: Instruction reads (executed).
- **I1mr**: L1 instruction cache misses.
- **ILmr**: Last-level instruction cache misses.
- **Dr**: Data reads.
- **D1mr**: L1 data cache misses.
- **DLmr**: Last-level data cache misses.
- **Dw**: Data writes.
- **D1mw**: L1 data write misses.
- **DLmw**: Last-level data write misses.

In this example, 50% of data reads miss L1 (`D1mr` = 2,000 out of `Dr` = 4,000 for the `a[i]` read). The data is too large for L1 or has poor locality. `cg_annotate` also shows per-function and per-file summaries.

## Branch Simulation

Cachegrind's branch predictor simulator:

```bash
valgrind --tool=cachegrind --branch-sim=yes ./program
```

Adds columns:
- **Bc**: Conditional branches executed.
- **Bcm**: Conditional branches mispredicted.
- **Bi**: Indirect branches executed.
- **Bim**: Indirect branches mispredicted.

If `Bcm/Bc` exceeds 5%, your conditional branches are unpredictable. If `Bim/Bi` exceeds 5%, your virtual function calls or function pointer calls are unpredictable.

## Cachegrind Configuration

Cachegrind can simulate different cache configurations:

```bash
valgrind --tool=cachegrind \
    --I1=32768,8,64 \   # L1 instruction: 32KB, 8-way, 64B lines
    --D1=32768,8,64 \   # L1 data: 32KB, 8-way, 64B lines
    --LL=8388608,16,64  # L3 (LLC): 8MB, 16-way, 64B lines
    ./program
```

The defaults approximate your host CPU's cache. But Cachegrind only simulates two levels (L1 and LLC). There is no L2 simulation — the model is simplified. For more detailed simulation, use system simulators (gem5, Sniper) — but they are much slower.

## When to Use Cachegrind

Cachegrind excels at:
- **Deterministic, repeatable measurements** (no run-to-run variation).
- **Line-level attribution** (exactly which source line caused misses).
- **Algorithm comparison** (does algorithm A or B generate more cache misses for the same input?).

Cachegrind is weak at:
- **Real-world performance prediction** (the simulation is simplified; real CPUs have prefetchers, out-of-order execution, and non-blocking caches that Cachegrind doesn't model).
- **Speed** (20–50× slowdown; unsuitable for large workloads).
- **L2 cache effects** (only two cache levels simulated).

Use Cachegrind to *understand* cache behavior, then use `perf stat` on real hardware to *measure* it.

## Simulation vs. Measurement

A common error: "Cachegrind says 0.1% cache miss rate, so my code is fine" — but `perf stat` shows 30% LLC misses. Why?

1. Cachegrind's default cache parameters may not match your hardware.
2. Cachegrind doesn't model prefetching. Real hardware prefetchers eliminate many L1/L2 misses that Cachegrind counts.
3. Cachegrind doesn't model contention (multiple cores sharing L3).
4. Cachegrind doesn't model TLB misses.

The reverse error: "Cachegrind says 50% L1 miss rate, this code is terrible" — but real hardware with a good prefetcher has 5% effective miss rate.

**Always validate simulation results against hardware counters.** Cachegrind tells you *potential* problems; `perf stat` tells you *actual* problems.
