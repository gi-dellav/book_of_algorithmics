# Cache Lines

The fundamental unit of data transfer between caches and memory is the **cache line** — 64 bytes on all modern x86 CPUs. Understanding this single number explains a surprising range of performance phenomena.

## The Experiment

```rust
let n: usize = 64 * 1024 * 1024;  // 64M elements
let mut a: Vec<i32> = vec![0; n];

// Access every element with stride S
let mut stride = 1;
while stride <= 256 {
    let mut i = 0;
    while i < n {
        a[i] += 1;  // One read-modify-write per stride elements
        i += stride;
    }
    stride *= 2;
}
```

If stride = 1, we access every element: a[0], a[1], a[2], ... → scan the array.
If stride = 2, we access every other element: a[0], a[2], a[4], ...
If stride = 64, we access a[0], a[64], a[128], ... → one access per cache line.

Result: the time for strides 1 through 16 is roughly **identical**. At stride = 16 (64 bytes / 4 bytes = 16 ints per cache line), we still access one element per cache line — and the cache line was already loaded by the first access. The hardware loads the entire 64-byte line on the first access to any byte in that line. The remaining 15 accesses hit the cache.

At stride = 32 (128 bytes between accesses), time roughly doubles — we're accessing every other cache line, so half the cache lines are loaded but never fully used.

## Why 64 Bytes?

Cache line size is a tradeoff:
- **Larger lines**: Better spatial locality (more neighboring data loaded for free). Higher bandwidth (fewer address transactions per byte). But: more wasted bandwidth if data isn't used (overfetch), more false sharing between cores.
- **Smaller lines**: Less overfetch, less false sharing. But: more cache tag overhead (tags per byte), lower bandwidth.

64 bytes settled as the industry standard. ARM, x86, and most RISC-V implementations use 64-byte lines. Some specialized processors (GPUs, DSPs) use 128-byte lines because their workloads are more spatially regular.

## Practical Implications

1. **Pack hot data together.** If you access fields A and B together, put them in the same structure, ideally within the same 64-byte block.

2. **Align hot structures to cache line boundaries.** `alignas(64) struct HotData { ... };` ensures the structure doesn't straddle two cache lines, which would double the number of cache misses for accesses that span the boundary.

3. **Pad to avoid false sharing.** Two threads incrementing different fields of the same structure that happen to be on the same cache line will fight over ownership of that line (cache coherence ping-pong). Add padding: `char _pad[64];` between per-thread fields, or use `std::hardware_destructive_interference_size` (C++17).

4. **Sequential access is cache-efficient.** A linear scan reads each cache line exactly once. `for (i = 0; i < n; i++) sum += a[i];` reads n/16 cache lines (for ints), each containing 16 useful elements.

5. **Random access wastes cache capacity.** A random access pattern may read one byte from a cache line and never touch the other 63 bytes. The effective cache size for random access is the same as for sequential, but the *useful data per cache line* drops from 64 bytes to 8 (or less).

## Cache Line Prefetching

The CPU's hardware prefetcher detects sequential access patterns and fetches upcoming cache lines before they're requested. This is why sequential access is even faster than the cache line model predicts — the prefetcher hides the latency of all but the first few accesses.

When stride > 64 bytes AND stride is a power of two, the prefetcher may get confused (page boundary crossings, cache set conflicts). The associativity article explores this.
