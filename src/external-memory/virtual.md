# Virtual Memory

Virtual memory is the OS's abstraction that gives each process its own address space, isolated from other processes and from physical memory constraints. It's the mechanism behind `malloc`, `mmap`, and swap. It's also a performance minefield if you don't understand it.

## How Virtual Memory Works

Each process sees a flat 48-bit address space (256 TB on x86-64). The hardware (MMU — Memory Management Unit) translates virtual addresses to physical addresses transparently on every memory access.

Translation uses **page tables**: a 4-level radix tree (5-level with 57-bit addresses on newer CPUs). Each level indexes into a table; the final entry contains the physical page frame number + permissions.

- Page size: 4 KB (standard), 2 MB (huge pages), 1 GB (gigantic pages).
- 4-level page table: 4 × 9 = 36 bits of indexing → 4 KB × 512⁴ = 256 TB.

Every memory access would need 4 extra reads (the page table walk) if not for the TLB.

## The TLB (Translation Lookaside Buffer)

The TLB is a cache for virtual-to-physical address translations:

- **L1 TLB**: 64 entries (Zen 2), 1 cycle lookup.
- **L2 TLB**: 512 entries, ~5 cycle lookup.

A TLB hit: translation is free. A TLB miss: the hardware walks the page table (4 reads from memory — cached in the regular data cache, but still ~20 cycles). If any level of the page table is not in cache, the walk costs ~100+ cycles.

**TLB reach**: The L2 TLB covers 512 entries × 4 KB = 2 MB with standard pages. With 2 MB huge pages: 512 × 2 MB = 1 GB. If your working set exceeds TLB reach, you suffer TLB misses on top of cache misses.

## Huge Pages

Linux supports huge pages (2 MB) and gigantic pages (1 GB). Huge pages:
- Reduce TLB pressure (one TLB entry covers 512× more memory).
- Reduce page table depth (2 MB pages need only 3 levels instead of 4).
- Are transparently used via **Transparent Huge Pages** (THP) — the kernel coalesces contiguous 4 KB pages into 2 MB pages automatically.
- Can be explicitly allocated with `mmap(..., MAP_HUGETLB)` or `madvise(..., MADV_HUGEPAGE)`.

**Performance impact**: For applications with large, sequentially-accessed working sets (databases, in-memory caches), huge pages improve performance by 5–15% by reducing TLB miss rates. The `cpu-cache/paging.md` article has benchmarks.

## `mmap` for File I/O

```c
int fd = open("data.bin", O_RDONLY);
char *data = mmap(NULL, file_size, PROT_READ, MAP_PRIVATE, fd, 0);
// Now data[i] accesses the file contents via virtual memory
```

`mmap` maps a file into the process's address space. Reads from the mapped region trigger page faults; the kernel loads the corresponding file pages on demand. This can be more efficient than explicit `read()` calls because:
- No extra copy: the file data goes directly from the page cache to the process's address space.
- Lazy loading: only the pages that are actually accessed are loaded.
- The OS manages caching and prefetching.

But `mmap` has downsides:
- Page faults are expensive (~1 µs each). For sequential access of a large file, the fault overhead adds up.
- No control over prefetching (`madvise(MADV_SEQUENTIAL)` helps but is advisory).
- Complex error handling (SIGBUS if the file is truncated while mapped).

For sequential file I/O, `read()` with large buffers (64 KB+) is often faster than `mmap` + page faults. Both should be benchmarked for the specific workload.

## Swap

When physical memory is exhausted, the kernel moves some pages to disk (swap). The process that owns those pages is suspended while the data is retrieved — a "major page fault."

Swap costs ~1–10 ms (SSD) or ~10 ms (HDD) per fault. At 10 ms per fault, 100 faults = 1 second. A program that's "thrashing" (constantly faulting pages in and out) runs at 0.1% of its normal speed.

**Avoiding swap**:
- Reduce memory usage (smaller working sets, more compact data structures).
- Use `mlockall(MCL_CURRENT | MCL_FUTURE)` to pin critical pages in RAM.
- Set `vm.swappiness = 0` to discourage the kernel from swapping (but this doesn't prevent it entirely).
- Monitor `vmstat 1` — if `si` and `so` (swap in/out) are non-zero, you're swapping.

## Memory-Mapped I/O and DMA

The virtual memory system also supports mapping device memory into user space (e.g., GPU memory, FPGA registers, NVMe controller registers). The same page table mechanism translates CPU loads/stores into device accesses.

DMA (Direct Memory Access) allows devices to read/write physical memory directly, bypassing the CPU. The OS pins the pages (prevents swapping) and provides physical addresses to the device. This is how NVMe SSDs, network cards, and GPUs achieve high throughput without CPU involvement.
