# Multi-Core Sharing

Modern CPUs have multiple cores sharing caches, memory controllers, and interconnect bandwidth. When multiple threads access memory simultaneously, the performance effects are non-intuitive: bandwidth doesn't scale linearly, and false sharing can destroy performance.

## The Cache Hierarchy Revisited

On Zen 2 (Ryzen 7 4700U):
- **8 cores** in a single chip.
- **2 CCXes** (Core Complexes), each with 4 cores.
- **Per-core**: L1 (32 KB) + L2 (512 KB).
- **Per-CCX**: L3 (4 MB, shared by 4 cores).
- **System-wide**: 2×4 MB L3, connected via Infinity Fabric (~30–50 ns cross-CCX latency).
- **One memory controller** (single-channel DDR4 on this U-series SKU).

Key point: L3 is **not unified**. Cores 0–3 share one 4 MB L3; cores 4–7 share another. An access from core 0 to data in core 4's L3 goes across the Infinity Fabric — more expensive than a local L3 hit, cheaper than RAM.

## Cache Coherence

All cores must see a consistent view of memory. The **MESI protocol** (or variants: MESIF, MOESI) ensures coherence:

- **M**odified: This cache has the only valid copy. Must write back before eviction.
- **E**xclusive: This cache has the only copy, but it matches RAM. Can transition to Modified without a bus transaction.
- **S**hared: Multiple caches may have copies.
- **I**nvalid: This cache line is not present (or was invalidated by another core's write).

When core A writes to a cache line that core B has in Shared state, core B's copy is invalidated (transitioned to Invalid). B's next read of that line will miss and fetch the updated value from A's cache (or RAM).

This is the **cache coherence ping-pong**: if cores A and B alternate writing to the same cache line, the line bounces between their caches. Each bounce costs ~30–100 cycles (cross-CCX fabrics cost more). Throughput drops from one operation per cycle to one per 50+ cycles.

## False Sharing

Two threads increment different fields of the same structure — but the fields happen to be on the same cache line:

```rust
struct Counters {
    a: i32,  // Thread 0 increments this
    b: i32,  // Thread 1 increments this
}
// a and b are on the same cache line → false sharing!
```

Thread 0's increment writes the cache line, invalidating thread 1's copy. Thread 1's next increment must re-fetch the line. They ping-pong the line even though they access different data.

Fix: pad to cache line boundaries.

```rust
#[repr(C, align(64))]
struct Counters {
    a: i32,
    _pad1: [u8; 60],
    // b is now at offset 64 → different cache line
    b: i32,
}
```

`std::hardware_destructive_interference_size` (C++17) provides the minimum offset to avoid false sharing. Typically 64 on x86, 128 on some ARM systems.

## NUMA (Non-Uniform Memory Access)

On multi-socket servers, each socket has its own memory controller and local RAM. Accessing local RAM is fast (~100 ns); accessing remote RAM (the other socket's memory) is slower (~140–200 ns).

```bash
numactl --hardware   # Show NUMA topology
numactl --cpunodebind=0 --membind=0 ./program  # Pin to CPU node 0, memory node 0
lstopo                # Graphical topology view
```

For NUMA-aware programming:
- Allocate memory on the node where the accessing thread runs.
- Use `move_pages` to migrate pages to the accessing thread's node.
- Use `mmap(MAP_ANONYMOUS | MAP_HUGETLB)` + `madvise(MADV_HUGEPAGE)` for huge page allocation on the correct node.
- Libraries: `libnuma` provides `numa_alloc_onnode`, `numa_run_on_node`.

## Parallel Bandwidth Scaling

If one core saturates RAM bandwidth (~20 GB/s on single-channel DDR4-3200), two cores sharing the same memory channel will get ~10 GB/s each — **bandwidth is a shared resource**. On a dual-channel system, two cores may each get near-peak bandwidth if they're accessing different channels.

```bash
# Measure aggregate bandwidth with 1, 2, 4, 8 threads
for t in 1 2 4 8; do
    ./stream_bench -t $t
done
```

Results (dual-channel Zen 2 desktop):
- 1 thread: ~25 GB/s (one channel saturated)
- 2 threads: ~50 GB/s (both channels saturated)
- 4 threads: ~50 GB/s (saturated — no further gain)
- 8 threads: ~50 GB/s (bandwidth-bound — adding threads doesn't increase total throughput)

## CPU Affinity

Pinning threads to specific cores prevents the OS scheduler from migrating them, which would cause cold caches and TLB misses:

```bash
taskset -c 0-3 ./program  # Pin to cores 0, 1, 2, 3
```

```rust
use std::mem;
let mut cpuset: libc::cpu_set_t = unsafe { mem::zeroed() };
unsafe { libc::CPU_ZERO(&mut cpuset); }
unsafe { libc::CPU_SET(0, &mut cpuset); }
unsafe { libc::pthread_setaffinity_np(libc::pthread_self(), mem::size_of::<libc::cpu_set_t>(), &cpuset); }
```

For performance-critical multi-threaded code:
- Pin each thread to a specific core.
- Allocate thread-local data on the same NUMA node as the thread's core.
- Avoid oversubscription (more threads than cores).
- Consider hyperthreading: a physical core runs two logical threads, sharing execution units. Hyperthreading helps for memory-bound workloads (one thread can compute while the other is waiting for memory) but can hurt compute-bound workloads (two threads compete for the same execution units).
