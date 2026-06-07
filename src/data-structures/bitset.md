# Bitmap Structures

A bitmap (or bitset) represents a set of integers as a bit array: bit `i` is 1 if element `i` is present. Operations are O(1): set, clear, and test a bit. But the real power of bitmaps comes from their interaction with hardware: SIMD for bulk operations, `tzcnt` for fast iteration, and `popcount` for cardinality. Compressed bitmaps (Roaring) extend this to sparse sets with minimal memory.

## Why Bitmaps?

A `std::set<int>` for 1M elements costs ~32 MB (32 bytes per node). A bitmap for the same range [0, 10⁷) costs ~1.25 MB (one bit per integer). For dense sets (occupancy > 1%), bitmaps use less memory and are faster for union, intersection, and iteration. For sparse sets (occupancy < 0.1%), compressed formats are better.

## Basic Operations

```rust
fn bitmap_set(bitmap: *mut u64, x: usize) {
    unsafe {
        *bitmap.add(x >> 6) |= 1u64 << (x & 63);
    }
}

fn bitmap_clear(bitmap: *mut u64, x: usize) {
    unsafe {
        *bitmap.add(x >> 6) &= !(1u64 << (x & 63));
    }
}

fn bitmap_test(bitmap: *const u64, x: usize) -> bool {
    unsafe {
        ((*bitmap.add(x >> 6) >> (x & 63)) & 1) != 0
    }
}
```

Three instructions each: shift, AND/mask, and memory access. ~3ns per operation on Zen 2.

## Iteration: Finding Set Bits

Iterating over a bitmap by testing every bit takes O(range) time — terrible for sparse sets. Instead, skip whole 64-bit words of zeros:

```rust
fn bitmap_iterate(bitmap: *const u64, n_words: usize, f: fn(usize)) {
    unsafe {
        for i in 0..n_words {
            let mut word = *bitmap.add(i);
            while word != 0 {
                let bit = word.trailing_zeros() as usize;
                f(i * 64 + bit);
                word &= word - 1;
            }
        }
    }
}
```

`__builtin_ctzll` compiles to `tzcnt` (3-cycle latency, 1 per cycle throughput). The `word &= word - 1` clears the lowest set bit — alternatively `_blsr_u64(word)` is a single instruction (BLSR = reset lowest set bit, 1 cycle).

For very dense bitmaps (>50% ones), it's faster to iterate all positions and test each bit rather than use `tzcnt` (most `tzcnt` calls will return small skips). The crossover is around 50% density.

## Population Count (Cardinality)

```rust
fn bitmap_popcount(bitmap: *const u64, n_words: usize) -> u32 {
    let mut count: u32 = 0;
    unsafe {
        for i in 0..n_words {
            count += (*bitmap.add(i)).count_ones();
        }
    }
    count
}
```

`__builtin_popcountll` → `popcnt` instruction: 3-cycle latency, 1 per cycle throughput on Zen 2. For 1M elements (15,625 words): ~47,000 cycles → ~23.5 µs.

With AVX-512, `_mm512_popcnt_epi64` processes 8 64-bit words per instruction:

```rust
use std::arch::x86_64::*;

fn bitmap_popcount_avx512(bitmap: *const u64, n_words: usize) -> i32 {
    unsafe {
        let mut vsum = _mm512_setzero_si512();
        let mut i = 0;
        while i + 8 <= n_words {
            let v = _mm512_loadu_si512(bitmap.add(i) as *const __m512i);
            vsum = _mm512_add_epi64(vsum, _mm512_popcnt_epi64(v));
            i += 8;
        }
        let mut count = _mm512_reduce_add_epi64(vsum);
        while i < n_words {
            count += (*bitmap.add(i)).count_ones() as i32;
            i += 1;
        }
        count
    }
}
```

~4× faster than scalar popcount for large bitmaps.

## Set Operations: Union, Intersection, Difference

```rust
fn bitmap_union(dst: *mut u64, a: *const u64, b: *const u64, n_words: usize) {
    unsafe {
        for i in 0..n_words {
            *dst.add(i) = *a.add(i) | *b.add(i);
        }
    }
}
```

These compile to trivial SIMD loops. AVX2 processes 4 words per cycle. For 1M elements: ~15,000 words / 4 = ~3750 iterations → ~6 µs for a full union on Zen 2.

## Compressed Bitmaps: Roaring

For sparse sets, a dense bitmap wastes space. A set with 100 elements in [0, 10⁹) needs a 125 MB bitmap. The **Roaring bitmap** (Chambi et al., 2016) partitions the range into chunks of 2¹⁶ (65,536 integers) and stores each chunk as either:

- A **sorted array of 16-bit integers** (if the chunk has ≤ 4096 elements): 2 bytes per element.
- A **bitmap of 2¹⁶ bits = 8 KB** (if the chunk has > 4096 elements).

The threshold 4096 balances array and bitmap memory cost: 4096 × 2 bytes = 8 KB.

```rust
#[repr(C)]
struct RoaringContainer {
    key: u16,
    is_bitmap: bool,
    // union {
    //     *mut u16 sorted_array;
    //     [u64; 1024] bitmap;  // 2^16 bits
    // }
    data_ptr: *mut u8,
    cardinality: i16,
}

#[repr(C)]
struct RoaringBitmap {
    containers: *mut RoaringContainer,
    num_containers: i32,
}
```

Operations dispatch per container: two arrays merge, two bitmaps OR, array+bitmap iterates the array setting bits. Performance: union of two Roaring bitmaps (1M elements, 1% density, range [0, 10⁹)): ~2 ms vs. ~15 ms for a dense bitmap. The Roaring bitmap is ~7× faster and uses ~1% of the memory.

## Rank and Select

Beyond set membership, bitmaps support two advanced operations crucial for succinct data structures:

- **rank₁(i)**: number of set bits in positions [0, i).
- **select₁(j)**: the position of the j-th set bit (1-indexed).

**Rank with precomputed blocks**: divide into blocks of 512 bits, precompute cumulative popcount per block:

```rust
fn rank(bitmap: *const u64, block_rank: *const u32, i: usize) -> u32 {
    unsafe {
        let block = i / 512;
        let offset = i % 512;
        let word_start = block * 8 + (offset / 64);
        let bit_offset = offset % 64;
        let mut count = *block_rank.add(block);
        for w in (block * 8)..word_start {
            count += (*bitmap.add(w)).count_ones();
        }
        count += ((*bitmap.add(word_start)) & ((1u64 << bit_offset) - 1)).count_ones();
        count
    }
}
```

O(1): one block lookup + up to 7 popcounts. ~15ns on Zen 2.

**Select with multi-level indices**: binary search on blocks to find the block containing the j-th set bit, then scan within the block. Two-level indices (superblocks of 8192 bits + blocks of 512 bits) give O(1) select at the cost of additional precomputation.

Rank and select enable **succinct data structures**: compressed representations that still support O(1) queries. Applications: suffix arrays (FM-index), wavelet trees, and compressed graph representations that are ~10× smaller than uncompressed equivalents.

## SIMD Bit Iteration

For extracting all set bits, `tzcnt` per word is optimal. For bulk extraction from 4 words at once with an early skip:

```rust
use std::arch::x86_64::*;

fn bitmap_iterate_avx2(bitmap: *const u64, n_words: usize, f: fn(usize)) {
    unsafe {
        let mut i = 0;
        while i + 4 <= n_words {
            let v = _mm256_loadu_si256(bitmap.add(i) as *const __m256i);
            if _mm256_testz_si256(v, v) == 0 {
                let mut words: [u64; 4] = [0; 4];
                _mm256_storeu_si256(words.as_mut_ptr() as *mut __m256i, v);
                for j in 0..4 {
                    let mut w = words[j];
                    while w != 0 {
                        let bit = w.trailing_zeros() as usize;
                        f((i + j) * 64 + bit);
                        w &= w - 1;
                    }
                }
            }
            i += 4;
        }
        while i < n_words {
            let mut w = *bitmap.add(i);
            while w != 0 {
                let bit = w.trailing_zeros() as usize;
                f(i * 64 + bit);
                w &= w - 1;
            }
            i += 1;
        }
    }
}
```

`_mm256_testz_si256` checks if all 4 words are zero in one instruction, skipping empty chunks. For very sparse bitmaps (density < 5%), this is 2-4× faster than per-word checking.

## Performance Summary (Zen 2)

| Operation | 1M elements, 10% dense | 1M elements, 90% dense |
|-----------|----------------------|----------------------|
| Set membership | 3 ns | 3 ns |
| Iterate all set bits | 0.5 ms | 1.8 ms |
| Popcount | 0.02 ms | 0.02 ms |
| Union (two bitmaps) | 0.006 ms | 0.006 ms |
| Intersection | 0.006 ms | 0.006 ms |
| Roaring union (1% dense, range 10⁹) | 2 ms | N/A |

## Key Lessons

1. **Bitmaps dominate for dense sets.** O(1) operations, trivial vectorization, smallest memory footprint. The only competition is very sparse sets or sets with very large ranges.
2. **`tzcnt` + `blsr` is the optimal iteration pattern.** Each set bit visited in O(popcount) time with single-cycle instructions.
3. **Roaring bitmaps handle sparse sets gracefully.** The hybrid array/bitmap approach gives near-dense-bitmap speed at near-sparse-array memory. Used everywhere in big data (Lucene, Druid, Spark, Redis).
4. **Rank and select unlock succinct data structures.** With O(1) rank/select, you can build compressed suffix arrays and wavelet trees 10× smaller than uncompressed, with minimal query overhead.
5. **SIMD helps everywhere in bitmaps.** Bulk operations vectorize automatically. AVX2 skip-ahead accelerates sparse iteration. AVX-512 VPOPCNTQ accelerates popcount 4×.
