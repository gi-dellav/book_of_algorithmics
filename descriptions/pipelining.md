# Chapter: Instruction-Level Parallelism (`pipelining/`)

## Overview

Weight 3 in the book. This chapter covers the internal pipelined, superscalar, out-of-order execution model of modern CPUs. It explains pipeline hazards (structural, data, control), branch prediction and its costs, branchless programming via predication (`cmov`), instruction tables (latency/throughput), throughput computing with multiple accumulators, and touches on scheduling and theoretical limits. The `_index.md` uses an education analogy (students, classes, teachers) to explain pipelining, superscalar, and OoO execution.

## Files and Content

| File | Status | Size | Description |
|------|--------|------|-------------|
| `_index.md` | Complete | 6.1 KB | Pipelining, superscalar, and OoO execution explained via education analogy. CPI vs. latency distinction. |
| `branching.md` | Published | 6.5 KB | Cost of branching experiment (random threshold, ~14 cycles). Branch prediction quality vs. probability. Pattern detection (sorted vs. unsorted). `[[likely]]` hint effect. |
| `branchless.md` | Published | 11.3 KB | Predication: arithmetic trick `(a[i] - 50) >> 31`, `cmov`, ternary operator equivalence. When predication wins (threshold ~75%). Examples: empty string optimization, branchless binary search, data-parallel masking. |
| `hazards.md` | Published | 1.9 KB | Pipeline hazards: structural, data, control. Pipeline stalls and bubbles. |
| `limits.md` | Draft | 2.0 KB | Theoretical performance limits: decode width, memory bandwidth, sorting lower bound, FMA peak. Brief. |
| `scheduling.md` | Draft | 4.0 KB | Superscalar processors, μops, execution ports, out-of-order scheduling, instruction window. |
| `tables.md` | Complete | 3.4 KB | Instruction tables: latency vs. throughput definitions, sample Zen 2 values, notes on pipelining and variable latency (div). |
| `throughput.md` | Complete | 6.5 KB | Throughput vs. latency optimization. Multiple accumulators to saturate execution units. General formula: need `latency × throughput` accumulators. |

## Image Assets

5 images: `bubble.png` (pipeline stall), `pipeline.png` (pipeline diagram), `branchy-vs-branchless.svg` (performance crossover at ~75%), `probabilities.svg` (branch prediction vs. probability), `superscalar.png` (superscalar pipeline).

## Strengths

1. **`branching.md` is a classic**: The experiment showing branch cost vs. probability, with the famous "sorted array is faster" phenomenon, is the most accessible demonstration of branch prediction in existence. The 75% crossover point graph is excellent.
2. **`branchless.md` is practical**: Shows exactly how to eliminate branches (arithmetic trick → `cmov` → ternary), when it's beneficial (the 75% threshold), and real-world examples (branchless binary search, SIMD masking). The empty-string optimization anecdote is memorable.
3. **`throughput.md` is immediately useful**: The "multiple accumulators" technique — using `latency × throughput` accumulators to saturate execution units — is a bread-and-butter optimization that readers can apply immediately.
4. **Clear hazard taxonomy**: Structural/data/control hazards are cleanly defined and prioritized by severity.
5. **Good connection to `cpu-cache/`**: The throughput article references memory bandwidth limitations and the bandwidth-latency product from the cache chapter.

## Areas for Improvement

1. **`scheduling.md` is rough**: The outline-style writing and incomplete thoughts ("You can schedule independent instructions separately, but only up to some extent...") need polishing. The μop concept is introduced but not fully explained.
2. **`limits.md` is too brief**: At 2.0 KB, it only sketches the idea of theoretical limits without providing concrete numbers or a methodology for calculating them. The "roofline model" is never mentioned by name.
3. **No unified example across articles**: Each article uses different code snippets. A single running example (e.g., optimizing a simple loop through hazard elimination, branch removal, and throughput tuning) would tie the chapter together.
4. **Missing: out-of-order execution depth**: The `_index.md` mentions OoO but no article explains the reorder buffer (ROB), register renaming, or the limits of OoO window size.
5. **Missing: dependency chains and critical path analysis**: While `throughput.md` mentions critical paths, there's no systematic method for drawing dependency graphs and identifying the critical path in a code snippet.
6. **`tables.md` doesn't reference tools**: Mention `llvm-mca`, `uiCA`, or Agner Fog's tables as practical resources for looking up latencies/throughputs.

## Recommendations

1. **Polish `scheduling.md`**: Explain the reorder buffer, register renaming, and the instruction window. Include a concrete example of how the CPU might schedule a short instruction sequence.
2. **Expand `limits.md`**: Introduce the roofline model (compute-bound vs. memory-bound regions), show how to calculate peak FLOPS and peak bandwidth, and provide a roofline plot for the Zen 2 test system.
3. **Add a "Critical Path Analysis" article**: Show how to draw a dependency graph for a code snippet, identify the critical path, and estimate minimum execution time from instruction latencies.
4. **Add a running example**: Take one loop (e.g., the sum-with-threshold from `branching.md`) through the entire optimization pipeline: eliminate the control hazard (branchless), eliminate the data hazard (multiple accumulators), and check against decode/retirement limits.
5. **Add tools references**: Link `tables.md` to `uops.info`, Agner Fog's tables, and `llvm-mca` for practical latency/throughput lookup.
6. **Consider merging `limits.md` with `throughput.md`**: The two articles are closely related. A combined article on "Throughput Limits and Roofline Analysis" would be more coherent.
