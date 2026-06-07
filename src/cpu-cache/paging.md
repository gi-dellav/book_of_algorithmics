# TLBs and Paging

The Translation Lookaside Buffer (TLB) caches virtual-to-physical address translations. When the TLB misses, the hardware performs a page table walk — 4 sequential memory accesses that each may miss the cache. TLB behavior is a hidden dimension of memory performance.

## The TLB Hierarchy (Zen 2)

| TLB | Entries | Coverage (4 KB pages) | Coverage (2 MB pages) |
|-----|---------|----------------------|----------------------|
| L1 DTLB | 64 | 256 KB | 128 MB |
| L1 ITLB | 64 | 256 KB | 128 MB |
| L2 TLB (unified) | 512 | 2 MB | 1 GB |

## The Strided Access Experiment

```rust
let mut step = 1;
while step <= 4096 {
    let mut i = 0;
    while i < n {
        // SAFETY: i is within bounds
        unsafe { *a.as_mut_ptr().add(i) += 1; }  // Access one element every 'step' ints
        i += step;
    }
    step *= 2;
}
```

For `step = 1` (sequential), each 4 KB page is accessed before moving to the next → TLB misses are minimized (one miss per page accessed). For `step = 4096` (16 KB between accesses), each access is to a different page → TLB misses on every access.

Result: at step = 4096, performance drops by ~20% for in-cache data (TLB miss penalty on top of cache hit) and by ~5% for RAM data (RAM latency dominates, hiding the TLB miss cost).

## TLB Reach

The "reach" of the TLB is the total memory covered by TLB entries:
- L2 TLB + 4 KB pages: 512 × 4 KB = 2 MB.
- L2 TLB + 2 MB huge pages: 512 × 2 MB = 1 GB.

If your working set > TLB reach, you suffer TLB misses. For a 16 GB data structure accessed with some irregularity, the TLB may thrash — evicting entries that will soon be needed again.

## Huge Pages: Measurement

```rust
// madvise to request huge pages for this allocation
use std::ptr;
let p = unsafe {
    libc::mmap(
        ptr::null_mut(),
        size,
        libc::PROT_READ | libc::PROT_WRITE,
        libc::MAP_PRIVATE | libc::MAP_ANONYMOUS | libc::MAP_HUGETLB,
        -1,
        0,
    )
};
```

Streaming through 16 GB with 4 KB pages vs. 2 MB huge pages:
- **4 KB pages**: ~15% TLB miss overhead (TLB misses generate page table walks that hit the L2/L3 cache but still cost ~20 cycles each).
- **2 MB pages**: ~0.1% TLB miss overhead (the entire working set fits in a fraction of the L2 TLB).

For sequential access, the benefit is modest (the prefetcher handles page table walks). For random access within a large working set, huge pages can improve performance by 20–50% by eliminating TLB thrashing.

Transparent Huge Pages (THP) attempt to use huge pages automatically. On Linux:
```bash
echo always > /sys/kernel/mm/transparent_hugepage/enabled
```

THP works well for anonymous mappings but can cause latency spikes when the kernel compacts memory to create a huge page. For performance-critical applications, explicit huge page allocation is more reliable.

## Page Table Structure

x86-64 uses a 4-level page table (5-level with LA57):
```
Virtual address: [16 unused][9 PGD][9 PUD][9 PMD][9 PTE][12 offset]
```

Each level is a 512-entry table (9 bits = 512 indices). The final PTE contains the physical page number + protection bits. A page table walk requires reading each level's entry sequentially — 4 dependent loads.

The page table entries themselves can be cached in the regular data cache. A TLB miss that hits in L1/L2 cached page tables costs ~20–30 cycles. A miss that goes to RAM costs ~100+ cycles.

## Large Pages vs. Huge Pages

- **4 KB**: Default. Fine granularity, large page table overhead.
- **2 MB** ("huge pages"): 512× larger. One PMD entry maps directly to a 2 MB page (no PTE level). Page table overhead is 1/512th.
- **1 GB** ("gigantic pages"): One PUD entry maps to 1 GB (no PMD or PTE). Reserved for very large, static allocations.

The tradeoff: larger pages reduce TLB pressure and page table depth, but increase internal fragmentation (a 2 MB page can only be used for one purpose; if you need 4 KB of it, 2 MB minus 4 KB is wasted).

## `madvise` and Memory Hints

```rust
unsafe { libc::madvise(ptr, size, libc::MADV_SEQUENTIAL); }    // Will be accessed sequentially → prefetch aggressively
unsafe { libc::madvise(ptr, size, libc::MADV_RANDOM); }        // Will be accessed randomly → don't prefetch
unsafe { libc::madvise(ptr, size, libc::MADV_HUGEPAGE); }      // Please use huge pages for this region
unsafe { libc::madvise(ptr, size, libc::MADV_DONTNEED); }      // I won't need this soon → evict from cache
unsafe { libc::madvise(ptr, size, libc::MADV_FREE); }          // I'm done with this → zero-fill on next access
```

These hints help the kernel manage memory more efficiently. `MADV_SEQUENTIAL` is particularly effective for large streaming reads — it tells the kernel to increase the read-ahead window and use non-temporal-like eviction.
