# Cache Bandwidth

Bandwidth is the rate at which data flows between cache levels. While latency limits single-thread pointer chasing, bandwidth limits bulk data processing: streaming through arrays, copying memory, and vector operations.

## The Experiment

Read (or write) an array of N integers sequentially, measuring throughput:

```c
for (int i = 0; i < n; i++)
    sum += a[i];  // Read-only bandwidth
```

Vary N from 2 KB to 256 MB, measure throughput in GB/s:

```
Array Size    Bandwidth (GB/s)    Explanation
-----------   ----------------   -----------
< 32 KB        ~200              L1 bandwidth (2 loads/cycle × 2 GHz × 64 bytes)
32–512 KB      ~100              L2 bandwidth
512 KB–8 MB    ~50               L3 bandwidth
> 8 MB          ~20              Single-channel DDR4-3200 (25.6 GB/s theoretical)
```

L1 bandwidth is enormous — ~200 GB/s on a single Zen 2 core. This is enough to read the entire L1 cache (32 KB) in ~160 ns. The limiting factor is the 2 load ports (2 × 64 bytes × 2 GHz = 256 GB/s theoretical).

L2 bandwidth is about half of L1. L3 is about half of L2. RAM is about half of L3 — but RAM bandwidth is shared across all cores, so a multi-threaded memory-bound workload saturates RAM bandwidth quickly.

## Directional Access

Not all accesses consume the same bandwidth:

| Direction | L1 Bandwidth | RAM Bandwidth | Notes |
|-----------|-------------|---------------|-------|
| Read only | Highest | ~20 GB/s | Pure load |
| Write only | Highest | ~10 GB/s | Write-allocate: the CPU reads the cache line before writing, costing 2× |
| Read + Write | Medium | ~15 GB/s | RMW operations access each line twice |

**Non-temporal stores** (`_mm256_stream_si256`) bypass the cache for writes, eliminating the read-for-ownership at L1/L2/L3. They're useful when writing large buffers that won't be read again soon. For RAM-to-RAM copies, non-temporal stores can achieve near-peak RAM bandwidth.

```c
for (int i = 0; i < n; i += 8) {
    __m256i data = _mm256_load_si256((__m256i*)(src + i));
    _mm256_stream_si256((__m256i*)(dst + i), data);  // Non-temporal store
}
```

## The Bandwidth Cliff

The bandwidth vs. array size graph shows sharp cliffs at cache boundaries. Unlike the latency experiment (which uses random access), the bandwidth experiment uses sequential access — and the hardware prefetcher kicks in. Even at sizes exceeding L3 capacity, the prefetcher fetches data ahead of the CPU, hiding some latency.

But the prefetcher can't increase bandwidth. Once you exceed a cache level's capacity, you're limited by the *next level's* bandwidth, not its latency. This is the key difference between latency-bound and bandwidth-bound workloads.

## Bandwidth-Delay Product

The product of latency and bandwidth gives the amount of data "in flight" at saturation:

- L1: 4 cycles × 200 GB/s = 800 bytes in flight (~12 cache lines)
- L3: 40 cycles × 50 GB/s = 2,000 bytes (~31 cache lines)
- RAM: 200 cycles × 20 GB/s = 4,000 bytes (~62 cache lines)

To saturate RAM bandwidth, the CPU needs ~62 cache lines in flight simultaneously. This is achieved through **memory-level parallelism** (see `mlp.md`) — the CPU's ability to overlap multiple cache misses. Without MLP, a single-thread streaming loop would achieve much lower bandwidth (limited by latency: one load at a time, 200 cycles each = 16 B / 200 cycles = 0.16 GB/s).

The OoO engine automatically overlaps memory accesses by looking ahead in the instruction stream and dispatching loads whose results aren't yet needed. The prefetcher helps by issuing loads even before the CPU requests them. Together, they keep enough loads in flight to saturate the memory bus.
