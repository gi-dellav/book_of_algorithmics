# Prefetching

Prefetching loads data into cache before the CPU requests it. Hardware prefetchers detect patterns automatically; software prefetching uses explicit instructions to hint the hardware. Both are essential for hiding memory latency.

## Hardware Prefetching

Modern CPUs have multiple hardware prefetchers that watch the stream of cache misses and predict future accesses:

- **Next-line prefetcher**: After accessing cache line L, prefetch L+1.
- **Stride prefetcher**: Detect constant-stride patterns (e.g., accessing every 256 bytes) and prefetch ahead along that stride.
- **Region prefetcher** (Zen 2): After accessing a "region" (aligned block), prefetch adjacent regions.

The prefetchers operate at L1, L2, and L3. L1 prefetchers fetch into L1; L2 prefetchers fetch into L2 (which is inclusive of L1 on Zen 2). They're aggressive but bounded — typically prefetching 4–16 cache lines ahead.

Hardware prefetching explains why sequential access achieves high bandwidth: the first few accesses trigger misses, but the prefetcher catches up and the rest hit the cache.

## When Hardware Prefetching Fails

1. **Random access**: No detectable pattern → prefetcher does nothing.
2. **Irregular strides**: Linked lists, hash tables, graph traversals.
3. **Very large strides (> 4 KB)**: Exceeds the region the prefetcher tracks.
4. **Too many streams**: If the program accesses more independent streams than the prefetcher can track (typically 8–16), some streams won't be prefetched.

## Software Prefetching

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::{_mm_prefetch, _MM_HINT_T0};

for i in 0..n {
    // Prefetch data needed 16 iterations ahead
    unsafe { _mm_prefetch(&a[i + 16] as *const _ as *const i8, _MM_HINT_T0) };  // Prefetch into L1
    process(a[i]);
}
```

`_mm_prefetch(addr, hint)` hints:
- `_MM_HINT_T0`: Prefetch into L1 (closest to CPU).
- `_MM_HINT_T1`: Prefetch into L2.
- `_MM_HINT_T2`: Prefetch into L3.
- `_MM_HINT_NTA`: Non-temporal access (treat as streaming, evict after use).

Software prefetching is a **hint** — the CPU may ignore it (if the address is already in cache, or if the memory subsystem is overloaded). It's not a load: it doesn't fault on invalid addresses and doesn't stall the pipeline.

## The Prefetch Distance Experiment

How far ahead should you prefetch? Too close → the data hasn't arrived by the time you need it. Too far → the data evicts something else before you use it, or arrives and is evicted before use.

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::{_mm_prefetch, _MM_HINT_T0};

let mut d = 1;
while d <= 64 {
    for i in 0..(n - d) {
        unsafe { _mm_prefetch(&a[i + d] as *const _ as *const i8, _MM_HINT_T0) };
        sum += a[i];
    }
    d *= 2;
}
```

The optimal D is roughly `latency / (access_time_per_element)`. If each element takes 10 cycles to process and RAM latency is 200 cycles, D = 20 is optimal — the data arrives just as we're ready to use it.

On Zen 2, optimal D for a simple accumulation loop is ~8–16 cache lines ahead. Larger D offers no benefit (and can hurt by evicting useful data). The prefetcher's built-in lookahead is usually sufficient for sequential access; software prefetching helps most for irregular but predictable patterns.

## D-Ahead Prefetching

For binary search in a sorted array (or Eytzinger layout), the access pattern is a tree descent — predictable if you know the comparison results, but not sequential. The data structures chapter (`data-structures/binary-search.md`) shows how to software-prefetch both children of each node:

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::{_mm_prefetch, _MM_HINT_T0};

while lo < hi {
    let mid = (lo + hi) / 2;
    unsafe {
        _mm_prefetch(&a[(lo + mid) / 2] as *const _ as *const i8, _MM_HINT_T0);  // Left grandchild
        _mm_prefetch(&a[(mid + hi) / 2] as *const _ as *const i8, _MM_HINT_T0);   // Right grandchild
    }
    if target < a[mid] {
        hi = mid;
    } else {
        lo = mid + 1;
    }
}
```

By prefetching two levels ahead, the data for the next two comparisons is in cache by the time we need it. This turns a latency-bound binary search (each step waits for the next cache line) into a partially bandwidth-bound one.

## Non-Temporal Prefetching

`_MM_HINT_NTA` hints that the data is "non-temporal" — it should be evicted after use, not kept in cache. This is useful for large streaming workloads where caching the data would evict more important data.

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::{__m256i, _mm256_load_si256, _mm256_stream_si256, _mm_prefetch, _MM_HINT_NTA};

for i in (0..n).step_by(8) {
    unsafe {
        _mm_prefetch(&src[i + 64] as *const _ as *const i8, _MM_HINT_NTA);
        let data = _mm256_load_si256(&src[i] as *const _ as *const __m256i);
        _mm256_stream_si256(&mut dst[i] as *mut _ as *mut __m256i, data);
    }
}
```

Combined with non-temporal stores, this achieves near-peak RAM-to-RAM copy bandwidth without polluting the caches.

## Pitfalls

1. **Prefetching already-cached data is wasted work.** The `_mm_prefetch` instruction occupies execution resources even if the data is in cache.
2. **Prefetching invalid addresses is fine** (it's a hint, not a load). But if the address would cause a page fault, the prefetch is silently dropped — don't rely on it to pre-fault pages.
3. **Over-prefetching evicts useful data.** If you prefetch 100 cache lines but only use 10, you've wasted 90 cache lines of capacity.
4. **The hardware prefetcher is usually smarter than you** for sequential access. Software prefetching is for irregular patterns the hardware can't detect.
