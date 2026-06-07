# Experimental Cache Characterization

The external memory model tells you *why* caches matter. This chapter shows you *how much* they matter — on real hardware, with real benchmarks. Every claim is backed by a microbenchmark you can run on your own machine.

## The Test Platform

All experiments in this chapter were performed on a Ryzen 7 4700U (Zen 2, 8 cores, 2.0 GHz base / 4.1 GHz boost):

| Cache | Size | Latency | Associativity | Line Size |
|-------|------|---------|---------------|-----------|
| L1 Data | 32 KB (per core) | 4 cycles | 8-way | 64 B |
| L1 Instruction | 32 KB (per core) | 4 cycles | 8-way | 64 B |
| L2 | 512 KB (per core) | 12 cycles | 8-way | 64 B |
| L3 (LLC) | 8 MB (shared, 2×4MB CCX) | ~40 cycles | 16-way | 64 B |
| RAM (DDR4-3200) | 16 GB | ~100 ns (~200 cycles) | — | 64 B (burst) |

All benchmarks use a single core pinned with `taskset -c 0`, frequency locked to 2.0 GHz (`cpupower frequency-set -f 2.0GHz`), with huge pages enabled where noted.

## Methodology

This chapter follows a deliberate structure: each article introduces one concept, demonstrates it with an experiment, shows the performance graph, and explains the result. The experiments build on each other — latency measurement requires understanding cache lines; bandwidth measurement builds on latency; associativity builds on both.

The experiments are designed to be **reproducible**. Code is provided (C, compiled with `-O2`). Parameters (cache sizes, line sizes, associativity) are reported so you can adapt the benchmarks to your own hardware.

## What You'll Learn

By the end of this chapter, you will have:
- Measured every cache level's latency and bandwidth on your own machine.
- Seen the associativity cliffs — and learned to avoid power-of-two strides.
- Understood when to use Array of Structures vs. Structure of Arrays.
- Learned to use hardware and software prefetching.
- Measured memory-level parallelism and understood its limits.
- Seen the impact of TLBs and huge pages.
- Understood NUMA and multi-core cache effects.

This chapter is the practical companion to the external memory model. Where Chapter 8 gives you asymptotic analysis, this chapter gives you concrete numbers and the intuition to apply the theory.

## Acknowledgements

This chapter draws heavily on Igor Ostrovsky's "Gallery of Processor Cache Effects" (2010) and Ulrich Drepper's "What Every Programmer Should Know About Memory" (2007). The experiments have been updated for modern hardware (Zen 2), but the methodology and many of the insights are timeless.
