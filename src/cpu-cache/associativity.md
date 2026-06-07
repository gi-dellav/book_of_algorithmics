# Cache Associativity

Caches are not fully associative — a given memory address can only be stored in a specific subset of cache lines. This constraint, called **associativity**, causes surprising performance cliffs when accessing data at power-of-two strides.

## Set-Associative Caches

A cache is divided into **sets**, each containing **W** ways (typically 8 for L1/L2, 16 for L3). A memory address maps to exactly one set (determined by bits of the address), but can be stored in any of the W ways within that set.

```
Address decomposition (L1: 32 KB, 8-way, 64 B lines):
  bits 0-5:   Cache line offset (6 bits → 64 bytes)
  bits 6-11:  Set index (6 bits → 64 sets)
  bits 12-63: Tag (identifies which address is in this way)

64 sets × 8 ways × 64 bytes = 32 KB
```

## The Experiment

```rust
for stride in 1..=256 {
    let mut i = 0;
    while i < n {
        a[i] += 1;  // Access one element per stride
        i += stride;
    }
}
```

**Expected**: Time should be proportional to 1/stride (fewer accesses at larger strides).

**Actual**: At stride = 64 (64 ints × 4 bytes = 256 bytes), timing spikes — 10× slower than stride = 63 or 65!

Why? Stride 64 means consecutive accesses are 256 bytes apart. 256 bytes = 4 cache lines. The set index bits are the same for addresses 256 bytes apart (256 / 64 = 4, and 4 is a power of two → bits 6-7 don't change). All accesses map to the same 8 sets (out of 64), thrashing the cache.

At stride 63 (252 bytes), consecutive accesses map to different sets, using the full cache effectively.

At stride 65 (260 bytes), same — no power-of-two conflict.

## The Power-of-Two Problem

Any power-of-two stride (in bytes) causes associativity conflicts:
- Stride 64 bytes (1 cache line): all accesses to the same set.
- Stride 128 bytes (2 cache lines): all accesses to 2 sets.
- Stride 4 KB (page size): all accesses to the same set (4 KB / 64 bytes = 64 cache lines = 64 sets exactly).
- Stride 1 MB: all accesses to the same set.

This is why hash tables should use prime-number sizes (or at least non-power-of-two sizes). A hash table with 2^20 slots accessed by a bad hash function might only use a few cache sets, effectively reducing the L1 cache from 32 KB to ~500 bytes.

## The Fenwick Tree Example

A Fenwick tree stores elements at indices `i += i & -i` (add the lowest set bit). For an array of size 2^20, the parent of node i is at `i + (i & -i)`. The gaps between successive accesses in an update operation are powers of two.

Result: Fenwick tree operations trigger associativity conflicts. The fix: insert "holes" in the array (unused elements) so that the stride is not a power of two. Adding a small constant offset to each index breaks the pattern. The data structures chapter (`data-structures/segment-trees.md`) covers this in detail.

## Associativity Misses vs. Capacity Misses

A **capacity miss** occurs when the working set exceeds the cache size — the cache is full, and something must be evicted. No amount of clever layout can fix a capacity miss (except reducing the working set).

An **associativity miss** occurs when the working set fits in cache but maps to too few sets. The data is evicted not because the cache is full, but because all accesses compete for the same sets. Fix: change the access pattern (strides, alignment) or add padding.

## Practical Rules

1. **Avoid power-of-two strides in multi-dimensional arrays.** A 1024×1024 matrix (power-of-two rows) accessing columns has stride 4096 = 64 × 64 → associativity collisions. Add padding: `float A[1024][1024 + 16]` — the extra 64 bytes shift each row's starting offset.

2. **Use odd block sizes for tiled algorithms.** Block size 64 is a power of two; 63 or 65 avoids associativity issues.

3. **Prime hash table sizes** not only improve hash distribution — they also avoid cache associativity conflicts.

4. **Randomize allocation bases.** Address space layout randomization (ASLR) helps by randomizing the base address of heap allocations, spreading them across cache sets.
