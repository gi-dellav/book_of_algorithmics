# Cache Latency

Cache latency is the time to service a load that misses all higher cache levels and hits at this level. Measuring it requires the **pointer chasing** technique — creating a linked list that forces the CPU to follow a chain of dependent loads, eliminating the possibility of overlap.

## The Pointer Chasing Experiment

Create an array of indices where each entry points to the next entry, forming a random permutation cycle:

```rust
let n: usize = 64 * 1024 * 1024;  // Array size in bytes
let num_indices = n / std::mem::size_of::<i32>();
let mut a: Vec<i32> = vec![0; num_indices];
let mut perm: Vec<usize> = (0..num_indices).collect();

// Create random permutation
// shuffle(&mut perm);  // Use rand crate: perm.shuffle(&mut rng)

// Chain pointers: each element points to the next in the permutation
for i in 0..num_indices - 1 {
    a[perm[i]] = perm[i + 1] as i32;
}
a[perm[num_indices - 1]] = perm[0] as i32;

// Measure pointer chasing
let mut x: std::sync::atomic::AtomicI32 = std::sync::atomic::AtomicI32::new(0);
let mut idx = perm[0];
for _ in 0..ITERS {
    // SAFETY: idx is always within bounds
    idx = unsafe { *a.as_ptr().add(idx) } as usize;  // Dependent load → pure latency measurement
    x.store(idx as i32, std::sync::atomic::Ordering::Relaxed);
}
```

The `volatile` and the dependency through `idx` prevent the CPU from overlapping iterations. Each iteration does exactly one load, which must complete before the next can begin. Divide total cycles by ITERS to get the average latency.

## The Latency Curve

```
Array Size    Latency (cycles)    Explanation
-----------   -----------------   -----------
< 32 KB       4                   L1 data cache hit
32 KB – 512 KB  12                L2 cache hit
512 KB – 8 MB   40                L3 cache hit (local CCX)
> 8 MB          200               RAM access (DDR4-3200)
```

The plateau transitions are sharp — within a factor of 2 of the cache size. This is because the random permutation means every access is a cache miss when the working set exceeds the cache's capacity. There's no prefetching (the accesses are unpredictable), no spatial locality (one int per cache line).

## Frequency Scaling Effect

If you run the same experiment at different CPU frequencies, the latency in *cycles* stays roughly the same for cache hits (the cache is synchronous with the core). But RAM latency in *cycles* increases with frequency — a 100 ns RAM access is 200 cycles at 2 GHz but 300 cycles at 3 GHz. This is why overclocking doesn't improve memory-bound workloads as much as compute-bound ones.

## Latency vs. Bandwidth

The pointer-chasing experiment measures **latency** — the minimum time for a single load in isolation. It achieves near-zero **bandwidth** — one load followed by 4+ cycles of waiting.

The bandwidth experiment (next article) measures the opposite: many loads in flight simultaneously, saturating the memory bus. Together, they bound performance: for a given workload, your access time is at least max(latency_per_access, bytes_per_access / bandwidth).

## The Memory Wall

The ratio of RAM latency to CPU cycle time has grown from ~1:1 (1980s) to ~200:1 (today). While CPU clock speeds increased 100×, RAM latency in nanoseconds improved by only ~5× (from ~100 ns to ~50 ns, with some back-and-forth as DDR generations traded latency for bandwidth).

This is the **memory wall**: the CPU spends an ever-larger fraction of its time waiting for memory. Caches hide this for well-behaved workloads, but any algorithm that chases pointers through a large data structure is limited by RAM latency — not CPU speed, not algorithmic complexity, but the physical time for electrons to travel from the CPU to the DRAM chip and back.
