# Performance Pitfalls

You've parallelized your code. You're using 8 threads. But the speedup is only 2×. What went wrong? This article catalogs the common reasons parallel performance disappoints — and how to fix them.

## 1. Amdahl's Law: The Serial Fraction

If 10% of your program is serial, maximum speedup is 1/0.10 = 10×, regardless of how many cores you have. At 100 cores, efficiency is 10%. At 1000 cores, it's 1%.

**Diagnosis**: profile with 1 thread, then with N threads. Compute the serial fraction: `s = (1/speedup - 1) / (1/N - 1)`. If s is high, find what part of the program isn't parallelized.

**Fix**: identify the serial bottleneck. Common culprits: I/O (reading a single input file), initialization (building data structures before spawning threads), and final reduction (combining results). Move I/O to parallel reads, build data structures in parallel when possible, and use parallel reduction.

## 2. False Sharing

Two threads write to different variables that happen to be in the same cache line (64 bytes). The cache coherence protocol (MESI) ping-pongs the line between cores, even though they're accessing different data.

```c
struct {
    int counter_a;  // Thread A increments this
    int counter_b;  // Thread B increments this
    // counter_a and counter_b are 4 bytes apart → same cache line
} shared;

// Performance: each increment invalidates the other thread's cache line.
// Throughput drops from 1 increment/7ns to 1 increment/50ns.
```

**Diagnosis**: `perf stat -e cache-misses,cache-references` shows high cache miss rates that don't correlate with the working set size. Or: add padding and see if performance improves.

**Fix**: pad shared variables to 64-byte alignment:

```c
struct alignas(64) PaddedCounter { int value; };
PaddedCounter counter_a;
PaddedCounter counter_b;
```

Or in C++: `alignas(std::hardware_destructive_interference_size)`.

## 3. Lock Contention

Threads spend more time waiting for locks than doing useful work. A single hot mutex serializes the entire program.

**Diagnosis**: `perf record -e sched:sched_switch` to see context switches. If threads frequently sleep on the same mutex, it's a bottleneck. Or use mutex profiling tools.

**Fix**:
- **Shrink critical sections**: move work outside the lock.
- **Shard data**: instead of one mutex for a hash table, use one mutex per bucket or per stripe.
- **Use lock-free data structures**: atomics instead of mutexes for simple operations.
- **Batch operations**: acquire the lock once, do multiple operations, release.

## 4. Load Imbalance

One thread finishes its work and sits idle while others are still busy. The total time is determined by the slowest thread.

**Diagnosis**: timeline visualization (Intel VTune, Chrome Tracing). If threads have different completion times, it's load imbalance.

**Fix**:
- **Dynamic scheduling**: instead of statically assigning `N/P` iterations to each thread, use work stealing (each thread grabs work from a shared queue).
- **Guided scheduling (OpenMP)**: `schedule(guided)` assigns larger chunks first, smaller chunks later to balance load.
- **Shuffle the input**: if some data elements take longer to process, randomize the assignment so that no thread gets all the "hard" elements.

## 5. Over-Subscription

More threads than hardware threads. The OS time-slices between them, adding context switch overhead. CPU-bound threads should never exceed the number of logical cores.

**Diagnosis**: `htop` shows more active threads than CPUs. Context switch rate from `perf stat -e context-switches` is high (> 10,000/s).

**Fix**: use a thread pool sized to `std::thread::hardware_concurrency()`. For CPU-bound work, exactly this many threads. For IO-bound work, up to 2-4× this many (to overlap I/O with computation).

## 6. NUMA Effects

On multi-socket machines, each socket has its own memory controller. Accessing memory attached to the "wrong" socket costs ~1.5× latency and half the bandwidth. If all threads are on socket 0 but the data is on socket 1, performance suffers.

**Diagnosis**: `numactl --hardware` to see NUMA topology. `numastat -p <pid>` to see per-node memory allocation.

**Fix**:
- **First-touch policy**: the kernel allocates memory on the NUMA node that first touches a page. Initialize data in parallel: the thread that will use a page should touch it first.
- **`numactl --interleave=all`**: interleave memory across all nodes. Good for shared data. Bad for thread-local data (increases remote accesses).
- **`numactl --membind=0 --cpunodebind=0`**: pin process to a single NUMA node. If your working set fits on one node, this eliminates NUMA effects entirely.

## 7. Memory Bandwidth Saturation

Parallel computation may be memory-bound. One thread saturates 50% of memory bandwidth. Two threads saturate it. Four threads fight for it — no additional speedup.

**Diagnosis**: `perf stat -e cycles,instructions` shows low IPC (< 1) even though the algorithm is "compute-heavy." `likwid-perfctr` or Intel PCM to measure memory bandwidth.

**Fix**: improve cache utilization (blocking, tiling, prefetching). If bandwidth is the limit, more threads won't help — the chip is already at its DRAM bandwidth limit.

## 8. Thread Startup Cost

Creating and joining threads per parallel region adds ~15 µs overhead per region. If your parallel region runs in 10 µs, you're losing money.

**Diagnosis**: the program is slower with threading than without for small workloads.

**Fix**: use a thread pool, keep threads alive, and only fork-join for large workloads (> 100 µs). OpenMP does this automatically (threads spin-wait after a parallel region).

## 9. False Optimism from Hyperthreading

Hyperthreading (SMT) makes one physical core look like two logical cores. But the two threads share execution units, L1 cache, and memory bandwidth. A parallel program may show "speedup" from 8 to 16 threads on a 8-core machine, but the per-thread throughput drops. The speedup from 8→16 is usually 20-50%, not 100%.

**Diagnosis**: run with hyperthreading disabled (kernel boot parameter `nosmt`) and compare. Or compare performance at `N` threads vs. `N/2` threads on N logical cores.

**Fix**: for CPU-bound code, use `num_physical_cores` threads, not `num_logical_cores`. For IO-bound code, hyperthreading helps (more threads to hide I/O latency).

## Performance Checklist

| Symptom | Probable Cause | Fix |
|---------|---------------|-----|
| Speedup << N, high single-thread performance | Amdahl's law (serial fraction) | Parallelize the serial part |
| Speedup plateaus at 2-3×, high cache misses | False sharing | Align/pad shared data |
| Speedup plateaus, high context switch rate | Lock contention | Shard, lock-free, shrink critical sections |
| Threads finish at different times | Load imbalance | Dynamic scheduling, work stealing |
| Speedup decreases beyond N threads | Over-subscription | Thread pool, hardware_concurrency() |
| Speedup limited even with low IPC | Memory bandwidth saturation | Improve cache utilization |
| Multi-socket worse than single-socket | NUMA effects | First-touch initialization, `numactl` |
