# Computation Models

The Random Access Machine is not the only model of computation — it's just the one most algorithm textbooks use. When the RAM model's assumptions break down (and they always break down on real hardware), other models can provide better predictive power.

## The Word RAM Model

The standard RAM model assumes each memory cell stores an arbitrary integer and each operation costs O(1). The **word RAM model** constrains this: each memory cell stores w bits, and operations on w-bit words cost O(1). The word size w is typically 32 or 64.

This matters because operations that are O(1) in the standard model may not be O(1) in reality. Multiplying two n-bit integers takes O(n²) time with the grammar-school algorithm, O(n^1.585) with Karatsuba, and O(n log n) with FFT-based methods. The standard model hides this complexity; the word RAM model surfaces it when n exceeds w.

## The External Memory Model

The most important alternative model for this book. Also called the **I/O model** or **disk access model** (DAM), it was formalized by Aggarwal and Vitter in 1988.

Parameters:
- **N**: problem size (in blocks)
- **M**: internal memory size (in blocks)
- **B**: block size (elements per block)

The model assumes:
- Computation on data in internal memory is free.
- Moving a block between external and internal memory costs 1 I/O.
- The algorithm controls which blocks are brought in and evicted.

The goal is to minimize the number of I/O operations. A "scan" of N elements costs SCAN(N) = ⌈N/B⌉ I/Os. Sorting N elements (external merge sort) costs SORT(N) = Θ((N/B) log_{M/B}(N/B)) I/Os.

The external memory model directly applies to CPU caches: M is the cache size, B is the cache line size (64 bytes on x86). The model also applies to RAM-to-disk I/O, where B might be 4 KB and M might be gigabytes.

Chapter 8 develops this model in detail. The key insight: algorithms designed for the external memory model automatically exploit cache locality, because moving a cache line costs 1 I/O while operating on the entire line's worth of data costs 0.

## The Parallel RAM (PRAM) Model

The PRAM model assumes P processors sharing a common memory. All processors execute the same program synchronously. Memory access is unit-cost and conflict-free (or handled by one of several contention-resolution conventions).

Variants:
- **EREW** (Exclusive Read, Exclusive Write): no two processors may access the same location simultaneously.
- **CREW** (Concurrent Read, Exclusive Write): multiple readers, exclusive writers.
- **CRCW** (Concurrent Read, Concurrent Write): anything goes, with a rule for write conflicts.

PRAM is useful for analyzing the *inherent parallelism* of an algorithm — the work (total operations) and span (longest dependency chain). But it ignores communication costs, synchronization overhead, and memory contention, which dominate real parallel performance.

Brent's theorem: any PRAM algorithm with work W and span S can be executed on P processors in time ≤ S + (W − S)/P. This gives an upper bound, but it's optimistic because it ignores scheduling overhead.

Chapter 13 covers parallel computing models and their connection to real hardware.

## Communication Complexity

In the two-party communication model, Alice holds input x, Bob holds input y, and they want to compute f(x, y). They exchange messages. The cost is the total number of bits communicated.

This model captures the inherent cost of distributed computation: if a function requires Ω(n) bits of communication, no distributed algorithm can be fast. Communication complexity lower bounds translate directly into lower bounds for data structures (e.g., the cell-probe model) and streaming algorithms.

## The Roofline Model

A practical model for understanding performance on real hardware. Plot operations per second against operational intensity (operations per byte of memory traffic). The "roofline" is formed by two ceilings:

- **Peak compute**: the processor's maximum FLOPS (horizontal line).
- **Peak bandwidth**: the memory system's maximum throughput (diagonal line, since FLOPS = intensity × bandwidth).

An algorithm's performance is bounded by whichever ceiling it hits. If operational intensity is low, the algorithm is memory-bound — adding more compute capability won't help. If intensity is high, it's compute-bound — better memory won't help.

The roofline model is a quick way to determine *which kind* of optimization to pursue. We use it in Chapter 3 (pipelining/limits) and return to it throughout the case studies.

## Quantum Computation

A brief note, since this model is fundamentally different: quantum computation uses qubits (superpositions of 0 and 1) and unitary transformations. The complexity classes change — BQP (bounded-error quantum polynomial time) contains problems believed to be outside P, including integer factorization (Shor's algorithm, O((log n)³)) and unstructured search (Grover's algorithm, O(√n)).

At the time of writing, practical quantum computers are too small and error-prone to outperform classical computers on useful problems. But the existence of quantum algorithms with proven exponential speedups means that some cryptographic assumptions (RSA, ECC) will eventually need to be replaced. Chapter 7 discusses post-quantum cryptography.

## Energy Models

An emerging concern: computation costs energy. In mobile devices, data centers, and edge computing, the relevant metric might be operations per joule rather than operations per second. The energy cost of an operation depends on: the operation type (add vs. multiply vs. divide), the data source (register vs. cache vs. RAM), and the voltage/frequency state of the processor.

Approximate numbers for a 45 nm CMOS process:
- 8-bit integer add: ~0.1 pJ
- 32-bit integer multiply: ~3 pJ
- 32-bit float multiply: ~4 pJ
- 32-bit SRAM read (8 KB): ~5 pJ
- 32-bit DRAM read: ~640 pJ

A single off-chip memory access consumes as much energy as ~160 integer additions. Energy optimization looks a lot like cache optimization — keep data local, access it sequentially, avoid recomputation of values that can be cached. The same principles that produce fast code tend to produce energy-efficient code.

## Which Model Should You Use?

For day-to-day optimization, the external memory model + the roofline model will cover 90% of cases. Start there. Use the word RAM model when dealing with big integers or bit manipulation. Pull in communication complexity when reasoning about distributed or multi-core code. And remember that all models are wrong; the test of a model is whether it leads you to faster code, not whether it captures every detail of the hardware.
