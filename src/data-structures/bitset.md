# Bitmap Structures

A bitmap (or bitset) represents a set of integers as a bit array: bit `i` is 1 if element `i` is present. Operations are O(1): set, clear, and test a bit. But the real power of bitmaps comes from their interaction with hardware: SIMD for bulk operations, `tzcnt` for fast iteration, and `popcount` for cardinality. Compressed bitmaps (Roaring) extend this to sparse sets with minimal memory.

## Why Bitmaps?

A `std::set<int>` for 1M elements costs ~32 MB (32 bytes per node). A bitmap for the same range [0, 10⁷) costs ~1.25 MB (one bit per integer). For dense sets (occupancy > 1%), bitmaps use less memory and are faster for union, intersection, and iteration. For sparse sets (occupancy < 0.1%), compressed formats are better.

## Basic Operations

```c
void bitmap_set(uint64_t *bitmap, int x) {
    bitmap[x >> 6] |= 1ULL << (x & 63);
}

void bitmap_clear(uint64_t *bitmap, int x) {
    bitmap[x >> 6] &= ~(1ULL << (x & 63));
}

bool bitmap_test(uint64_t *bitmap, int x) {
    return (bitmap[x >> 6] >> (x & 63)) & 1;
}
```

Three instructions each: shift, AND/mask, and memory access. ~3ns per operation on Zen 2.

## Iteration: Finding Set Bits

Iterating over a bitmap by testing every bit takes O(range) time — terrible for sparse sets. Instead, skip whole 64-bit words of zeros:

```c
void bitmap_iterate(uint64_t *bitmap, int n_words, void (*fn)(int)) {
    for (int i = 0; i < n_words; i++) {
        uint64_t word = bitmap[i];
        while (word) {
            int bit = __builtin_ctzll(word);  // Count trailing zeros → position
            fn(i * 64 + bit);
            word &= word - 1;  // Clear lowest set bit
        }
    }
}
```

`__builtin_ctzll` compiles to `tzcnt` (3-cycle latency, 1 per cycle throughput). The `word &= word - 1` clears the lowest set bit — alternatively `_blsr_u64(word)` is a single instruction (BLSR = reset lowest set bit, 1 cycle).

For very dense bitmaps (>50% ones), it's faster to iterate all positions and test each bit rather than use `tzcnt` (most `tzcnt` calls will return small skips). The crossover is around 50% density.

## Population Count (Cardinality)

```c
int bitmap_popcount(uint64_t *bitmap, int n_words) {
    int count = 0;
    for (int i = 0; i < n_words; i++)
        count += __builtin_popcountll(bitmap[i]);
    return count;
}
```

`__builtin_popcountll` → `popcnt` instruction: 3-cycle latency, 1 per cycle throughput on Zen 2. For 1M elements (15,625 words): ~47,000 cycles → ~23.5 µs.

With AVX-512, `_mm512_popcnt_epi64` processes 8 64-bit words per instruction:

```c
int bitmap_popcount_avx512(uint64_t *bitmap, int n_words) {
    __m512i vsum = _mm512_setzero_si512();
    int i = 0;
    for (; i + 8 <= n_words; i += 8) {
        __m512i v = _mm512_loadu_si512(&bitmap[i]);
        vsum = _mm512_add_epi64(vsum, _mm512_popcnt_epi64(v));
    }
    int count = _mm512_reduce_add_epi64(vsum);
    for (; i < n_words; i++)
        count += __builtin_popcountll(bitmap[i]);
    return count;
}
```

~4× faster than scalar popcount for large bitmaps.

## Set Operations: Union, Intersection, Difference

```c
void bitmap_union(uint64_t *dst, uint64_t *a, uint64_t *b, int n_words) {
    for (int i = 0; i < n_words; i++)
        dst[i] = a[i] | b[i];
}
```

These compile to trivial SIMD loops. AVX2 processes 4 words per cycle. For 1M elements: ~15,000 words / 4 = ~3750 iterations → ~6 µs for a full union on Zen 2.

## Compressed Bitmaps: Roaring

For sparse sets, a dense bitmap wastes space. A set with 100 elements in [0, 10⁹) needs a 125 MB bitmap. The **Roaring bitmap** (Chambi et al., 2016) partitions the range into chunks of 2¹⁶ (65,536 integers) and stores each chunk as either:

- A **sorted array of 16-bit integers** (if the chunk has ≤ 4096 elements): 2 bytes per element.
- A **bitmap of 2¹⁶ bits = 8 KB** (if the chunk has > 4096 elements).

The threshold 4096 balances array and bitmap memory cost: 4096 × 2 bytes = 8 KB.

```c
struct RoaringContainer {
    uint16_t key;              // High 16 bits of all elements
    bool is_bitmap;
    union {
        uint16_t *sorted_array;
        uint64_t bitmap[1024];  // 2^16 bits
    };
    int16_t cardinality;
};

struct RoaringBitmap {
    RoaringContainer *containers;
    int32_t num_containers;
};
```

Operations dispatch per container: two arrays merge, two bitmaps OR, array+bitmap iterates the array setting bits. Performance: union of two Roaring bitmaps (1M elements, 1% density, range [0, 10⁹)): ~2 ms vs. ~15 ms for a dense bitmap. The Roaring bitmap is ~7× faster and uses ~1% of the memory.

## Rank and Select

Beyond set membership, bitmaps support two advanced operations crucial for succinct data structures:

- **rank₁(i)**: number of set bits in positions [0, i).
- **select₁(j)**: the position of the j-th set bit (1-indexed).

**Rank with precomputed blocks**: divide into blocks of 512 bits, precompute cumulative popcount per block:

```c
int rank(uint64_t *bitmap, uint32_t *block_rank, int i) {
    int block = i / 512;
    int offset = i % 512;
    int word_start = block * 8 + (offset / 64);
    int bit_offset = offset % 64;
    int count = block_rank[block];
    for (int w = block * 8; w < word_start; w++)
        count += __builtin_popcountll(bitmap[w]);
    count += __builtin_popcountll(bitmap[word_start] & ((1ULL << bit_offset) - 1));
    return count;
}
```

O(1): one block lookup + up to 7 popcounts. ~15ns on Zen 2.

**Select with multi-level indices**: binary search on blocks to find the block containing the j-th set bit, then scan within the block. Two-level indices (superblocks of 8192 bits + blocks of 512 bits) give O(1) select at the cost of additional precomputation.

Rank and select enable **succinct data structures**: compressed representations that still support O(1) queries. Applications: suffix arrays (FM-index), wavelet trees, and compressed graph representations that are ~10× smaller than uncompressed equivalents.

## SIMD Bit Iteration

For extracting all set bits, `tzcnt` per word is optimal. For bulk extraction from 4 words at once with an early skip:

```c
void bitmap_iterate_avx2(uint64_t *bitmap, int n_words, void (*fn)(int)) {
    int i = 0;
    for (; i + 4 <= n_words; i += 4) {
        __m256i v = _mm256_loadu_si256((__m256i*)&bitmap[i]);
        if (_mm256_testz_si256(v, v) == 0) {  // Not all zeros
            uint64_t words[4];
            _mm256_storeu_si256((__m256i*)words, v);
            for (int j = 0; j < 4; j++) {
                uint64_t w = words[j];
                while (w) {
                    int bit = __builtin_ctzll(w);
                    fn((i + j) * 64 + bit);
                    w &= w - 1;
                }
            }
        }
    }
    for (; i < n_words; i++) {
        uint64_t w = bitmap[i];
        while (w) {
            int bit = __builtin_ctzll(w);
            fn(i * 64 + bit);
            w &= w - 1;
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
