# Elias-Fano Encoding

Elias-Fano encoding represents a sorted sequence of n integers from [0, u) in `n × ⌈log₂(u/n)⌉ + 2n` bits — within 2 bits per element of the information-theoretic optimum. And it supports O(1) access to the i-th element without decoding the whole sequence. This is the data structure underlying most production inverted indexes.

## The Representation

Split each value x into:
- **Low bits**: the lower ℓ = ⌈log₂(u/n)⌉ bits (stored explicitly).
- **High bits**: the upper ⌈log₂ u⌉ - ℓ bits (encoded as a bitvector of monotonically increasing values).

The high bits are encoded as a bitvector where, for each value x with high bits h, we set the bit at position h + i (where i is the position in the input). This is called the **unary encoding** of the high bits:

```rust
fn elias_fano_encode(values: &[u64], u: u64) -> (Vec<u64>, Vec<u64>) {
    let n = values.len() as u64;
    if n == 0 { return (vec![], vec![]); }

    let low_bits = if n >= u { 0 } else { (u as f64 / n as f64).log2().floor() as u32 };
    let low_mask = (1u64 << low_bits) - 1;

    let mut low_array = vec![0u64; ((n as usize) * low_bits as usize + 63) / 64];
    let mut high_bits = vec![0u64; ((n + u as usize) / 64) + 1];

    let mut low_bit_buf = 0u64;
    let mut low_buf_fill = 0u32;
    let mut li = 0usize;

    for (i, &v) in values.iter().enumerate() {
        let low = v & low_mask;
        let high = (v >> low_bits) as usize;

        // Write low bits
        low_bit_buf |= low << low_buf_fill;
        low_buf_fill += low_bits;
        while low_buf_fill >= 64 {
            low_array[li] = low_bit_buf as u64;
            li += 1;
            low_bit_buf >>= 64;
            low_buf_fill -= 64;
        }

        // Write high bits: set bit at position (high + i)
        let pos = high + i;
        high_bits[pos / 64] |= 1u64 << (pos % 64);
    }

    // Flush remaining low bits
    if low_buf_fill > 0 { low_array[li] = low_bit_buf as u64; }

    (low_array, high_bits)
}
```

The low bits array is a flat bit-packed array. The high bits bitvector has the property that there's exactly one 1 bit per element. The position of the i-th 1 bit is `high[i] + i`.

## O(1) Access

To retrieve the i-th element:

```rust
fn elias_fano_access(low_array: &[u64], high_bits: &[u64],
                      i: usize, low_bits: u32, low_mask: u64) -> u64 {
    // Extract low bits
    let bit_pos = i * low_bits as usize;
    let word_idx = bit_pos / 64;
    let bit_offset = bit_pos % 64;
    let low = if bit_offset + low_bits as usize <= 64 {
        (low_array[word_idx] >> bit_offset) & low_mask
    } else {
        // Crosses word boundary
        let part1 = low_array[word_idx] >> bit_offset;
        let part2 = low_array[word_idx + 1] << (64 - bit_offset);
        (part1 | part2) & low_mask
    };

    // Find the i-th 1 bit in high_bits → this gives high[i]
    // Use select operation: find position of i-th 1
    let high = select_1(high_bits, i);

    (high - i as u64) << low_bits | low
}
```

The `select_1` operation finds the position of the i-th 1 bit. This can be done in O(1) with a precomputed index that samples every S positions:

```rust
struct SelectIndex {
    bitvec: Vec<u64>,
    // Sample: every S bits, record cumulative count and position
    samples: Vec<(usize, usize)>, // (count of 1s, byte position)
}

fn select_1(idx: &SelectIndex, k: usize) -> usize {
    let sample = k / 512; // sample every 512 ones
    let (count, pos) = idx.samples[sample];
    // Linear scan from sample to find the exact bit
    let remaining = (k - count) as u32;
    let mut bit_pos = pos * 64;
    let mut ones = 0u32;
    for &word in &idx.bitvec[pos..] {
        let word_ones = word.count_ones();
        if ones + word_ones > remaining {
            let offset = select_in_word(word, remaining - ones);
            return bit_pos + offset as usize;
        }
        ones += word_ones;
        bit_pos += 64;
    }
    bit_pos
}
```

With sampling every 512 ones, the linear scan visits at most 8 words (512 bits / 64). The `select_in_word` uses `tzcnt` (trailing zero count) to find the k-th set bit in a word. Total: ~15 cycles per access.

## Elias-Fano vs. Alternatives

| Property | Varint | Simple16 | Elias-Fano |
|----------|--------|----------|------------|
| Space (Gov2 posting lists) | ~11 bits/int | ~3.5 bits/int | ~2.8 bits/int |
| Sequential decode | 4 GB/s | 5 GB/s | 2 GB/s |
| Random access (i-th element) | O(i) decode | O(block) then O(1) | O(1) |
| NextGEQ(x) query | O(n) scan | O(n) scan | O(log n) binary search |

Elias-Fano's killer feature is **NextGEQ** (next greater or equal): given value x, find the first element ≥ x. This is the core operation for boolean AND/OR on inverted indexes. With Elias-Fano, you can binary search on the high bits (using rank/select on the bitvector) and then scan within a narrow range. For two posting lists of lengths n₁ and n₂, intersection with Elias-Fano takes O(n₁ log n₂ / n₁) = O(n₁ log (n₂/n₁)) — sublinear when one list is much shorter.

## Practical Implementation: `sux-rs`

The `sux` crate provides a production-quality Elias-Fano implementation with optimized rank/select structures. For most applications, use it instead of rolling your own:

```rust
// use sux::dict::EliasFano;
// let ef = EliasFano::from_iter(values.iter().copied());
// let val = ef.get(i); // O(1) access
// let pos = ef.next_geq(x); // O(log n) search
```

## When Not to Use Elias-Fano

- **Unsorted data**: Elias-Fano requires monotonic input. Use dictionary coding or Frame-of-Reference.
- **Decode-only, bulk access**: SIMD bit-packing decodes faster for full-block reads.
- **Frequent updates**: Elias-Fano is static. For dynamic sequences, use an Elias-Fano-encoded B-tree (ε-τ trees or partitioned Elias-Fano with a mutable tail).
- **n > u/2**: When the sequence is dense, Elias-Fano overhead (2n bits) dominates. Use a plain bitmap instead.

## Benchmark (1M sorted document IDs)

| Operation | Elias-Fano | Varint-GB | Simple16 |
|-----------|-----------|-----------|----------|
| Size | 350 KB | 1070 KB | 820 KB |
| Sequential decode | 1.4 ms | 1.2 ms | 0.95 ms |
| Random access (single) | 18 ns | — | — |
| NextGEQ (100K queries) | 0.8 ms | 18 ms | 8.5 ms |
| Intersection (two 1M lists) | 1.9 ms | 12 ms | 6.3 ms |

For intersection-heavy workloads (the dominant pattern in search engines), Elias-Fano's random access and NextGEQ capabilities provide a 3–6× speedup over sequential-decode-only formats — and it does so in half the space.
