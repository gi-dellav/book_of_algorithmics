# Memory Management

Every program allocates memory. How it does so — through `malloc`, `new`, custom allocators, or memory pools — affects performance in ways that are easy to overlook. This article covers the practical side of memory allocation.

## The System Allocator

glibc's `malloc` (ptmalloc2) is the default on Linux. It's a general-purpose allocator optimized for a wide range of workloads:

```c
void *ptr = malloc(size);
free(ptr);
void *ptr2 = realloc(ptr, new_size);
void *ptr3 = calloc(count, size);  // Allocate and zero
```

Internally, ptmalloc uses:
- **Arenas**: Multiple independent allocation regions to reduce contention in multi-threaded programs. Each thread gets its own arena (up to a limit).
- **Bins**: Free lists of similarly-sized chunks. Small allocations (<512 bytes) go to fastbins (LIFO, no coalescing). Larger allocations go to regular bins (sorted by size, coalescing with neighbors).
- **`sbrk`/`mmap`**: Small arenas grow via `sbrk` (extending the data segment). Large allocations (>128 KB by default) use `mmap` directly (separate mappings that can be freed independently).

**Performance**: `malloc` of a small object takes ~20–50 cycles (fastbin hit) to ~200 cycles (need to extend the arena). `free` is similar. Thread contention on the arena lock adds significant overhead.

## Common Allocation Pitfalls

1. **Frequent small allocations**: `malloc(16)` in a loop at 1M iterations/second = 1M allocations/second. Each is a few dozen cycles, but the cumulative cost is significant. Use a memory pool or pre-allocate.

2. **Hidden allocations**: `std::vector::push_back` may reallocate and copy. `std::string` may allocate the string buffer. `std::function` may heap-allocate the callable. Know your abstractions' allocation behavior.

3. **Fragmentation**: Over time, free memory becomes scattered in small gaps that can't satisfy large allocation requests. The allocator spends more time searching for free space and may fail even when enough total memory is free.

4. **False sharing**: Two threads accessing different data that happens to be in the same cache line. The cache coherence protocol bounces the line between cores, killing performance. `malloc` may place allocations from different threads on the same cache line.

## Custom Allocators

When the general-purpose allocator doesn't meet your needs:

**Arena allocator**: Allocate a large block once, then "allocate" by bumping a pointer.
```c
typedef struct { char *base; char *current; char *end; } Arena;

void *arena_alloc(Arena *a, size_t size) {
    if (a->current + size > a->end) return NULL;
    void *ptr = a->current;
    a->current += size;
    return ptr;
}
```
Allocation is ~2 cycles (pointer bump + bounds check). Freeing is a no-op (the entire arena is freed at once). Perfect for per-frame allocations in game engines, per-request allocations in web servers, or any scoped computation.

**Pool allocator**: Allocate objects of a fixed size from pre-allocated slabs. No fragmentation, fast (free list: pop from list head). Used when you have many objects of the same type.

**Stack allocator**: Like arena, but with `free` that unwinds to a previously-saved marker. Good for temporary allocations with nested lifetimes.

**Slab allocator**: Pre-allocate pages and carve them into fixed-size slots. The kernel uses slab allocators for in-kernel objects (inodes, dentries, task structs). The principle applies in user space too.

## Alternative `malloc` Implementations

- **jemalloc** (FreeBSD, Facebook): Optimized for multi-threaded workloads and low fragmentation. Used in Redis, Rust, Firefox.
- **tcmalloc** (Google): Thread-caching malloc. Each thread has a local cache of free objects; bulk transfers between thread caches and central heap. Used in Chromium, gRPC.
- **mimalloc** (Microsoft): Compact, fast, and security-focused. Excellent performance on allocation-heavy benchmarks. Used in some Windows components.

Swapping `malloc` is as simple as `LD_PRELOAD=libjemalloc.so ./program`. Benchmark your workload with each — the differences can be 2–5× on allocation-heavy code.

## Garbage Collection

Managed languages (Java, Go, C#) use garbage collection instead of manual memory management. GC pauses — when the collector stops all application threads to trace live objects — are the primary performance concern.

Modern GCs are generational:
- **Young generation**: New objects, collected frequently (minor GC, fast). Most objects die young.
- **Old generation**: Objects that survive several minor GCs, collected rarely (major GC, slower).

The key tuning parameter: minimize the number of objects promoted to the old generation, and ensure major GC pauses are within the application's latency budget.

For C and C++, there is no GC (though Boehm GC exists, it's rarely used in performance-critical code). Instead, the programmer controls allocation and deallocation explicitly.
