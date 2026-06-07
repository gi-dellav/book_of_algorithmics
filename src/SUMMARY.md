# Summary

[Introduction](introduction.md)

# Complexity Models

- [Why Go Beyond Big O?](complexity/_index.md)
- [Hardware History](complexity/hardware.md)
- [Languages and Performance](complexity/languages.md)
- [Levels of Optimization](complexity/levels.md)
- [Computation Models](complexity/models.md)

# Computer Architecture

- [Learning Architecture First](architecture/_index.md)
- [ISA: RISC vs. CISC](architecture/isa.md)
- [Assembly Language](architecture/assembly.md)
- [Loops and Conditionals](architecture/loops.md)
- [Functions and the Stack](architecture/functions.md)
- [Indirect Branching](architecture/indirect.md)
- [Machine Code Layout](architecture/layout.md)
- [Interrupts and Syscalls](architecture/interaction.md)

# Instruction-Level Parallelism

- [Pipelining, Superscalar, OoO](pipelining/_index.md)
- [Pipeline Hazards](pipelining/hazards.md)
- [Branch Prediction](pipelining/branching.md)
- [Branchless Programming](pipelining/branchless.md)
- [Instruction Tables](pipelining/tables.md)
- [Throughput Computing](pipelining/throughput.md)
- [Scheduling and µops](pipelining/scheduling.md)
- [Theoretical Limits](pipelining/limits.md)

# Compilation

- [What the Compiler Does](compilation/_index.md)
- [Compilation Stages](compilation/stages.md)
- [Optimization Flags](compilation/flags.md)
- [Situational Optimizations](compilation/situational.md)
- [Contracts and Undefined Behavior](compilation/contracts.md)
- [Compile-Time Computation](compilation/precalc.md)
- [Zero-Cost Abstractions](compilation/abstractions.md)
- [Compiler Limitations](compilation/limitations.md)
- [Link-Time Optimization](compilation/lto.md)

# Profiling

- [The Three Tiers of Profiling](profiling/_index.md)
- [Instrumentation](profiling/instrumentation.md)
- [Statistical Profiling with perf](profiling/events.md)
- [Machine Code Analysis](profiling/mca.md)
- [Cache and Branch Simulation](profiling/simulation.md)
- [Benchmarking Methodology](profiling/benchmarking.md)
- [Noise and Bias](profiling/noise.md)

# Arithmetic

- [Darker Corners of the ISA](arithmetic/_index.md)
- [Integer Representations](arithmetic/integer.md)
- [IEEE 754 Floating Point](arithmetic/ieee-754.md)
- [Floating-Point Numbers in Depth](arithmetic/float.md)
- [Rounding Errors](arithmetic/errors.md)
- [Integer Division](arithmetic/division.md)
- [Newton's Method](arithmetic/newton.md)
- [Fast Inverse Square Root](arithmetic/rsqrt.md)
- [Bit Hacks](arithmetic/bit-hacks.md)
- [Fast Math Approximations](arithmetic/approx.md)
- [Data Compression](arithmetic/compression.md)

# Number Theory

- [The Uselessness that Changed the World](number-theory/_index.md)
- [Modular Arithmetic](number-theory/modular.md)
- [Binary Exponentiation](number-theory/exponentiation.md)
- [Extended Euclidean Algorithm](number-theory/euclid-extended.md)
- [Montgomery Multiplication](number-theory/montgomery.md)
- [Finite Fields](number-theory/finite.md)
- [Hash Functions](number-theory/hashing.md)
- [Cryptography](number-theory/cryptography.md)
- [Random Number Generation](number-theory/rng.md)
- [Error Correction](number-theory/error-correction.md)

# External Memory

- [The Cost of Data Movement](external-memory/_index.md)
- [Memory Hierarchy](external-memory/hierarchy.md)
- [The External Memory Model](external-memory/model.md)
- [Cache Eviction Policies](external-memory/policies.md)
- [Cache-Oblivious Algorithms](external-memory/oblivious.md)
- [Locality of Reference](external-memory/locality.md)
- [External Sorting](external-memory/sorting.md)
- [List Ranking](external-memory/list-ranking.md)
- [Virtual Memory](external-memory/virtual.md)
- [Memory Management](external-memory/management.md)
- [Sublinear Algorithms](external-memory/sublinear.md)

# RAM and CPU Caches

- [Experimental Cache Characterization](cpu-cache/_index.md)
- [Cache Lines](cpu-cache/cache-lines.md)
- [Latency](cpu-cache/latency.md)
- [Bandwidth](cpu-cache/bandwidth.md)
- [Associativity](cpu-cache/associativity.md)
- [Alignment and Padding](cpu-cache/alignment.md)
- [AoS vs. SoA](cpu-cache/aos-soa.md)
- [Memory-Level Parallelism](cpu-cache/mlp.md)
- [Prefetching](cpu-cache/prefetching.md)
- [TLBs and Paging](cpu-cache/paging.md)
- [Pointers and Alternatives](cpu-cache/pointers.md)
- [Multi-Core Sharing](cpu-cache/sharing.md)

# SIMD Parallelism

- [Data Parallelism on x86](simd/_index.md)
- [Auto-Vectorization](simd/auto-vectorization.md)
- [Intrinsics](simd/intrinsics.md)
- [Data Movement](simd/moving.md)
- [Masking and Blending](simd/masking.md)
- [Reductions](simd/reduction.md)
- [In-Register Shuffles](simd/shuffling.md)

# Algorithms Case Studies

- [Applied Algorithm Optimization](algorithms/_index.md)
- [Integer Factorization](algorithms/factorization.md)
- [Binary GCD](algorithms/gcd.md)
- [Argmin with SIMD](algorithms/argmin.md)
- [Prefix Sum](algorithms/prefix.md)
- [Matrix Multiplication](algorithms/matmul.md)
- [Logistic Regression Inference](algorithms/logistic.md)
- [Parsing Integers](algorithms/reading-integers.md)
- [Sorting](algorithms/sorting.md)

# Data Structures Case Studies

- [Multi-Dimensional Optimization](data-structures/_index.md)
- [Binary Search](data-structures/binary-search.md)
- [Static B-Trees](data-structures/s-tree.md)
- [Dynamic B-Trees](data-structures/b-tree.md)
- [Segment Trees](data-structures/segment-trees.md)
- [Hash Tables](data-structures/hash-tables.md)
- [Bitmap Structures](data-structures/bitset.md)
- [Bloom Filters](data-structures/filters.md)
- [Tree Structures Survey](data-structures/trees.md)

# Parallel Computing

- [Beyond the Single Core](parallel/_index.md)
- [Processes](parallel/concurrency/processes.md)
- [Threads](parallel/concurrency/threads.md)
- [Fibers and Goroutines](parallel/concurrency/fibers.md)
- [Event-Driven Concurrency](parallel/concurrency/event-driven.md)
- [Mutual Exclusion](parallel/synchronization/mutex.md)
- [Atomics and Lock-Free Programming](parallel/synchronization/atomics.md)
- [Memory Ordering](parallel/synchronization/memory-ordering.md)
- [Parallel Algorithms](parallel/algorithms/index.md)
- [OpenMP](parallel/algorithms/openmp.md)
- [GPU Programming](parallel/gpu/index.md)
- [CUDA](parallel/gpu/cuda.md)
- [Performance Pitfalls](parallel/pitfalls.md)
- [Threading Runtimes](parallel/runtimes.md)

# Distributed Computing

- [Computation Across Machines](distributed/_index.md)
- [Message Passing (MPI)](distributed/mpi/index.md)
- [Distributed Algorithms](distributed/algorithms/index.md)
- [MapReduce](distributed/mapreduce/index.md)
- [Actor Model](distributed/actor.md)
- [Cloud Computing](distributed/cloud.md)
