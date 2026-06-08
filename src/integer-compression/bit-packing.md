# Bit-Packing and SIMD

When all integers in a block are small (b bits each), we can pack them tightly: 32 integers of b bits into a single 32 × b-bit block. Decoding involves unpacking b-bit fields into 32-bit integers — and this is where SIMD shines.

## Naive Bit-Packing

Pack n integers of b bits each into `⌈n × b / 32⌉` 32-bit words:

```rust
fn pack(values: &[u32], bits: u32) -> Vec<u32> {
    let mut packed = Vec::new();
    let mut buf: u64 = 0;
    let mut buf_bits = 0u32;

    for &v in values {
        buf |= (v as u64) << buf_bits;
        buf_bits += bits;
        while buf_bits >= 32 {
            packed.push(buf as u32);
            buf >>= 32;
            buf_bits -= 32;
        }
    }
    if buf_bits > 0 { packed.push(buf as u32); }
    packed
}

fn unpack(packed: &[u32], bits: u32, count: usize) -> Vec<u32> {
    let mut values = vec![0u32; count];
    let mut buf: u64 = 0;
    let mut buf_bits = 0u32;
    let mut pi = 0usize; // position in packed

    for i in 0..count {
        while buf_bits < bits {
            buf |= (packed[pi] as u64) << buf_bits;
            buf_bits += 32;
            pi += 1;
        }
        values[i] = buf as u32 & ((1u32 << bits) - 1);
        buf >>= bits;
        buf_bits -= bits;
    }
    values
}
```

The inner loop of `unpack` has unpredictable branches (checking if `buf_bits < bits`). For n = 10⁶ and b = 13, decode takes ~2.3 ms — about 430 M integers/s. The bottleneck is the bit-shifting in the scalar loop.

## SIMD Bit-Unpacking: `_mm256_sllv_epi32`

The key SIMD instruction for unpacking is variable shift (`_mm256_sllv_epi32` in AVX2, `_mm512_sllv_epi32` in AVX-512). It shifts each 32-bit lane by a different amount:

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

unsafe fn unpack_avx2(packed: *const u32, bits: u32, values: *mut u32, count: usize) {
    let mask = (1u32 << bits) - 1;
    let mask_vec = _mm256_set1_epi32(mask as i32);

    // Precompute shifts for 32 / bits lanes
    // For bits = 5: shifts = [0, 5, 10, 15, 20, 25, 30, 3]
    let mut shifts = [0u32; 8];
    for i in 0..8 {
        let bit_pos = (i * bits) % 32;
        shifts[i] = bit_pos;
    }

    let mut pi = 0usize;
    let mut bit_buf = _mm256_setzero_si256();

    // Prime the bit buffer for staggered decode
    // ... (complex — see specialized implementations like TurboPack)

    for i in (0..count).step_by(8) {
        // SIMD right-shift and mask
        let shifted = _mm256_srlv_epi32(bit_buf, shift_vec);
        let values_vec = _mm256_and_si256(shifted, mask_vec);
        _mm256_storeu_si256(values.add(i) as *mut __m256i, values_vec);

        // Consume bits and refill
        let consumed = bits * 8;
        // ... (refill logic)
    }
}
```

The complexity of SIMD bit-unpacking is in managing the bit buffer across lane boundaries. Libraries like `simdcomp` (Daniel Lemire) and `TurboPFor` handle this. The speed payoff: ~8→12 GB/s decode throughput on Zen 2, or 2–3 billion integers/s.

## `_pext_u64` and `_pdep_u64` (BMI2)

Intel's BMI2 instruction set includes two instructions purpose-built for bit packing/unpacking:

- **`_pext_u64(src, mask)`**: Extract bits from `src` where `mask` has 1s, compacting them to the low bits.
- **`_pdep_u64(src, mask)`**: Deposit the low bits of `src` into positions where `mask` has 1s.

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::_pext_u64;

unsafe fn unpack_pext(packed: &[u64; 4], bits: u64) -> [u64; 64] {
    let mut result = [0u64; 64];
    let stride = bits as usize;
    let mut mask = (1u64 << bits) - 1;

    for i in 0..64 {
        let word_idx = (i * stride) / 64;
        let bit_pos = ((i * stride) % 64) as u64;
        let raw = _pext_u64(packed[word_idx] >> bit_pos, mask);
        result[i] = raw;
    }
    result
}
```

`_pext_u64` has 3-cycle latency and 1/cycle throughput on Zen 3, 1/cycle on Intel Haswell+. It extracts arbitrary bit fields in a single instruction. Combined with a loop over 64-bit words, it achieves ~15 GB/s decode — competitive with hand-tuned AVX2.

## Simple16 and PForDelta

**Simple16** packs as many integers as possible into a 28-bit payload (4-bit header indicates the packing scheme). For each block of 4–32 integers, choose the packing that wastes the fewest bits:

```
Header  Packing        Integers per 32-bit word
0000    1-bit  values   28
0001    2-bit  values   14
0010    3-bit  values    9
0011    4-bit  values    7
0100    5-bit  values    5
...
1111    28-bit values    1
```

Decode is a table lookup + SIMD unpack. Simple16 achieves ~2.5 bits per integer for typical posting lists.

**PForDelta** (Patched Frame of Reference) encodes most values with a small number of bits (say, 3 bits for 128 values = 384 bits), and stores "patches" (exceptions) separately for values that don't fit. Decode fills the block with the b-bit values, then overwrites the exception positions. This handles skewed distributions where most values are small but a few are large.

```rust
fn decode_pfordelta(packed: &[u32], block_size: usize, bits: u32,
                    exceptions: &[(usize, u32)]) -> Vec<u32> {
    let mut values = unpack(packed, bits, block_size);
    for &(pos, val) in exceptions {
        values[pos] = val;
    }
    values
}
```

PForDelta achieves ~2.3 bits per integer on web document posting lists while decoding at 5+ GB/s (using SIMD for the unpack, scalar for the patches).

## Choosing a Bit-Packing Scheme

| Scheme | Bits/int (typical) | Decode speed | Random access? |
|--------|-------------------|--------------|----------------|
| Varint-GB | 8–11 | 4 GB/s | No |
| Simple16 | 2.5–8 | 5 GB/s | No (block only) |
| PForDelta | 2.3–6 | 5 GB/s | No (block only) |
| SIMD bit-pack | User-chosen b | 8–12 GB/s | No |
| Elias-Fano | ~2 + log₂(n/m) | 2 GB/s | Yes (O(1)) |

For pure decode throughput, SIMD bit-packing at a fixed width is fastest. For good compression ratio with acceptable throughput, PForDelta. For theoretical optimality with random access, Elias-Fano (next article).

## Benchmark (1M sorted integers from Gov2 posting lists)

| Method | Encoded size | Decode time |
|--------|-------------|-------------|
| Delta + Varint-GB | 1.07 MB | 1.2 ms |
| Simple16 | 0.82 MB | 0.95 ms |
| PForDelta (b=3, 10% exceptions) | 0.78 MB | 0.88 ms |
| SIMD bit-pack (fixed b=13) | 1.58 MB | 0.31 ms |

PForDelta wins on the compression-speed Pareto frontier. The SIMD fixed-width pack is fastest but wastes bits on the few large values. In practice, choose based on whether you're I/O-bound (compress more) or CPU-bound (decode faster).
