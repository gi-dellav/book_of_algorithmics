# Frame-of-Reference and Dictionary Coding

Frame-of-Reference (FOR) is the workhorse of columnar databases and analytics engines. The idea: partition integers into blocks of 128–256 values, store the block's minimum, and then store each value as a `value - min` — a small delta that fits in fewer bits. Combined with SIMD bit-unpacking, FOR decodes at 10+ billion integers per second.

## Frame-of-Reference Encoding

```rust
fn encode_for(values: &[u32], block_size: usize, output: &mut Vec<u8>) {
    for block in values.chunks(block_size) {
        // Find the block minimum
        let min_val = *block.iter().min().unwrap_or(&0);
        output.extend_from_slice(&min_val.to_le_bytes());

        // Delta-encode: value - min_val
        let deltas: Vec<u32> = block.iter().map(|&v| v - min_val).collect();

        // Find max delta → determines bits needed
        let max_delta = *deltas.iter().max().unwrap_or(&0);
        let bits = 32 - max_delta.leading_zeros();
        output.push(bits as u8); // store bit width

        // Bit-pack the deltas
        pack(&deltas, bits, output);
    }
}
```

For a block of 128 timestamps in a log, the minimum is 1700000000 and values are within ±1000 of the minimum. Max delta = 1000 → 10 bits needed. Instead of storing 128 × 32 = 4096 bits, we store 32 (minimum) + 8 (bit width) + 128 × 10 = 1320 bits — a 3.1× reduction.

## SIMD Decode

Decoding FOR is SIMD's happy path: unpack b-bit integers, then add a broadcast minimum:

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

unsafe fn decode_for_avx2(input: &[u8], output: &mut [u32], block_size: usize) {
    // Read minimum
    let min_val = u32::from_le_bytes(input[0..4].try_into().unwrap());
    let bits = input[4] as u32;

    // Bit-unpack deltas into output
    unpack_avx2(input[5..].as_ptr() as *const u32, bits,
                output.as_mut_ptr(), block_size);

    // Add minimum: broadcast + vector add
    let min_vec = _mm256_set1_epi32(min_val as i32);
    for i in (0..block_size).step_by(8) {
        let deltas = _mm256_loadu_si256(output[i..].as_ptr() as *const __m256i);
        let values = _mm256_add_epi32(deltas, min_vec);
        _mm256_storeu_si256(output[i..].as_mut_ptr() as *mut __m256i, values);
    }
}
```

The `_mm256_add_epi32` has 0.5 cycle throughput per 8 integers — effectively free after the bit-unpacking.

## Adaptive FOR (AFOR)

When a block has outliers, FOR wastes bits on the max delta. AFOR handles this by splitting the block into two parts: values that fit in b bits, and exceptions:

```rust
fn encode_afor(values: &[u32], block_size: usize, output: &mut Vec<u8>) {
    for block in values.chunks(block_size) {
        let min_val = *block.iter().min().unwrap_or(&0);

        // Choose b such that 90% of values fit
        let mut deltas: Vec<u32> = block.iter().map(|&v| v - min_val).collect();
        deltas.sort_unstable(); // need order statistics

        let p90 = deltas[(block.len() as f64 * 0.9) as usize];
        let bits = 32 - p90.leading_zeros();
        let mask = (1u32 << bits) - 1;

        // Pack values that fit; mark exceptions
        let exception_mask: u64 = /* bitmap of exception positions */;
        // ... encode
    }
}
```

AFOR is essentially PForDelta (see the previous article) rebranded. It achieves near-optimal compression for skewed distributions while maintaining SIMD-friendly block sizes.

## Dictionary Coding

For columns with low cardinality (few distinct values), store a dictionary and encode values as dictionary indices:

```rust
fn encode_dictionary(values: &[u32], output: &mut Vec<u8>) -> Vec<u32> {
    // Build dictionary: distinct values
    let mut dict: Vec<u32> = values.to_vec();
    dict.sort_unstable();
    dict.dedup();

    // Encode as dictionary indices
    let indices: Vec<u32> = values.iter()
        .map(|v| dict.binary_search(v).unwrap() as u32)
        .collect();

    // Bit-pack indices (log2(dict.len()) bits each)
    let idx_bits = (dict.len() as f64).log2().ceil() as u32;
    pack_dictionary(&indices, idx_bits, output);

    dict
}
```

For a column of country codes (200 distinct values): 8 bits per value instead of 32 — a 4× reduction. For a column of product categories (10,000 distinct values): 14 bits per value — 2.3× reduction.

Dictionary coding combines seamlessly with FOR: dictionary-encode first, then delta-encode the indices within each block.

## FSST: Fast Static Symbol Table

For string columns, FSST (Boncz et al., 2018) builds a symbol table of frequent substrings and encodes strings as symbol sequences. It achieves 2–5× compression on text columns while decoding at 10+ GB/s:

```rust
// Pseudo-code — FSST uses a trained symbol table
struct FSST {
    symbols: Vec<[u8; 8]>,  // 1–8 byte symbols
    symbol_bits: u32,
}

fn fsst_decode(input: &[u8], fsst: &FSST, output: &mut Vec<u8>) {
    let mut pos = 0;
    while pos < input.len() {
        let code = /* extract symbol_bits bits */;
        let sym = &fsst.symbols[code];
        output.extend_from_slice(sym);
        pos += fsst.symbol_bits as usize;
    }
}
```

FSST is the string equivalent of dictionary coding for integers. Together with FOR and bit-packing, it forms the compression pipeline of systems like Apache Parquet, Apache Arrow, and DuckDB.

## The Full Compression Stack

A modern analytics database compresses a column in layers:

1. **Dictionary coding**: Map values to small integer codes.
2. **Delta coding**: Sort codes, store consecutive differences (if sorted).
3. **Frame-of-Reference**: Partition into blocks, encode delta from block minimum.
4. **Bit-packing**: Pack fixed-width integers into 32- or 64-bit words.

The decode chain reverses the order: bit-unpack → add frame minimum → cumulative sum (if deltas) → dictionary lookup.

## Benchmark Summary (1M 32-bit integers from different distributions)

| Distribution | Raw | Varint-GB | FOR | FOR + SIMD |
|-------------|-----|-----------|-----|------------|
| Sorted, dense (0..1M) | 4 MB | 1.0 MB | 0.5 MB | 0.5 MB |
| Sorted, gaps (Gov2 IDs) | 4 MB | 1.1 MB | 0.7 MB | 0.7 MB |
| Timestamps, ±1s jitter | 4 MB | 0.5 MB | 0.2 MB | 0.2 MB |
| Uniform random [0, 2³²) | 4 MB | 5 MB | 4.1 MB | 4.1 MB |

Decode throughput (FOR + SIMD): 10–15 GB/s, or 2.5–3.8 billion integers/s on Zen 2. This is within 2× of `memcpy` speed — you can't go much faster without moving the compression to the storage layer.

For production use, the `simdcomp` crate (Lemire et al.) and `stream-vbyte` provide battle-tested implementations of all the techniques in this chapter.
