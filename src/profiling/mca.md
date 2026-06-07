# Machine Code Analysis with `llvm-mca`

`llvm-mca` (LLVM Machine Code Analyzer) simulates a sequence of assembly instructions on a specific microarchitecture and reports expected throughput, port pressure, and critical path. It answers: "How fast *should* this code be, assuming perfect caches and no front-end bottlenecks?"

## What `llvm-mca` Does

Given assembly source and a target CPU:

1. Translates each instruction to the corresponding µops.
2. Simulates the out-of-order execution engine (ROB, scheduler, execution units).
3. Reports:
   - **Total cycles** (the simulation length).
   - **Instructions per cycle** (IPC).
   - **Block throughput** (cycles per iteration, for loops).
   - **Port pressure** (which execution ports are saturated).
   - **Critical path** (the longest dependency chain).

It assumes:
- All memory accesses hit the L1 cache (no misses).
- The branch predictor is perfect (no mispredictions).
- The fetch/decode stage has infinite bandwidth.

These assumptions make it an **upper bound** on performance. If your real code is slower than llvm-mca predicts, the difference is due to: cache misses, branch mispredictions, or front-end bottlenecks.

## Basic Usage

```bash
# Write your hot loop in a .s file (or extract with gcc -S)
llvm-mca -march=znver2 my_loop.s
```

Or inline assembly from C:
```bash
gcc -S -O2 -march=native my_loop.c -o - | llvm-mca -march=znver2
```

## Understanding the Output

Example: summing an array (AVX2, 8 floats at a time):

```asm
.Lloop:
    vaddps  ymm0, ymm0, [rdi]
    vaddps  ymm1, ymm1, [rdi + 32]
    add     rdi, 64
    cmp     rdi, rsi
    jne     .Lloop
```

`llvm-mca -march=znver2 -iterations=100`:

```
Iterations:        100
Instructions:      500
Total Cycles:      55
Total uOps:        400

Dispatch Width:    6
uOps Per Cycle:    7.27
IPC:               9.09
Block RThroughput: 0.5
```

Key numbers:
- **Block RThroughput: 0.5**: Each iteration can start every 0.5 cycles (2 iterations per cycle). The loop is throughput-bound, not latency-bound.
- **IPC: 9.09**: Much higher than the theoretical 4 instructions/cycle because µops are counted differently (fused loads, macro-fusion).

```
Resource pressure per iteration:
[0]    [1]    [2]    [3]    [4]    [5]    [6]    [7]
-      -     0.50   0.50    -      -     1.00    -
```

Ports 2 and 3 (loads) are at 0.5 each → 2 loads per 2 iterations → 1 load/cycle. Port 7 (store) is at 0 (this loop has no stores). The load ports are the bottleneck — 2 loads/iteration but only 2 load pipes available, and we issue 2 iterations per 0.5 cycle.

```
Critical sequence:
    vaddps ymm0, ymm0, [rdi]   (acc0 dependency chain)
        lat: 3
    vaddps ymm0, ymm0, [rdi]
        lat: 3
```

The individual accumulator chains are 3 cycles each, but since we have two independent accumulators, the critical path doesn't dominate — throughput does.

## Using `llvm-mca` to Find Bottlenecks

**Scenario 1: Port pressure is the bottleneck**

```
Resource pressure:
[0]  [1]  [2]  [3]  [4]  [5]  [6]  [7]
1.00 1.00 0.50 0.50  -    -    -    -
```

Ports 0 and 1 are at 1.0 — saturated. The loop uses too many integer/FP ALU operations for the available pipes. Solutions:
- Use SIMD to do more work per instruction.
- Use instructions that schedule to different ports.
- Accept the limit (the loop is compute-bound).

**Scenario 2: Dependency chain is the bottleneck**

```
Block RThroughput: 3.0
Critical sequence:
    vaddps ymm0, ymm0, ymm1   (3 cycles, feeds into itself)
```

The loop is latency-bound on a single accumulator. Solution: add more accumulators.

**Scenario 3: Neither — performance is limited by memory**

```
Block RThroughput: 0.25  (llvm-mca says it should be fast)
# But perf stat shows IPC = 0.5, high cache-misses
```

The loop should be fast, but real performance is limited by cache misses. llvm-mca assumes L1 hits. The gap between predicted and actual IPC is a measure of memory subsystem pressure.

## Integration with the Profiling Workflow

1. **`perf record` + `perf report`** → Find the hot function.
2. **`perf annotate`** → See the assembly of the hot loop.
3. **Copy the loop to a `.s` file** → Run `llvm-mca`.
4. **Compare predicted vs. actual IPC**:
   - If actual ≪ predicted: cache misses or branch mispredicts. Use `perf stat` to check.
   - If actual ≈ predicted but slow: hardware is at its limit. Algorithmic change needed.
   - If predicted is low (<1 IPC): the code is poorly scheduled. Try different instructions or unrolling.

## Limitations of `llvm-mca`

- Does NOT model the µop cache. Loops that exceed the µop cache capacity (~4096 µops) will see additional front-end stalls.
- Does NOT model cache misses. All memory accesses are assumed to hit L1.
- Does NOT model branch mispredictions. All branches are assumed perfectly predicted.
- The simulation is approximate. It may be off by 5–15% from real hardware for complex loops.

For more accurate analysis, use `uiCA` (uops.info Code Analyzer), which calibrates against real hardware measurements. For cache analysis, use Cachegrind or `perf stat -e cache-misses`.
