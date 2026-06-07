# In-Register Shuffles

Shuffles rearrange data within SIMD registers without touching memory. They're the Swiss Army knife of SIMD programming — used for transposition, lookup tables, data filtering, and permutations. Master shuffling and you master SIMD.

## The Primitives

### `pshufb` (SSSE3, 128-bit; AVX2, 256-bit)

The most versatile shuffle. Each byte of the destination is either:
- Copied from any byte of the source (indexed by the corresponding byte of the control mask, 0–127/255).
- Set to zero (if the high bit of the control byte is set).

```rust
let data = unsafe { _mm256_loadu_si256(...) };
let shuffle_mask = unsafe { _mm256_setr_epi8(
    3, 2, 1, 0,   // Reverse first 4 bytes
    7, 6, 5, 4,   // Reverse next 4 bytes
    -1, -1, -1, -1,  // Zero these 4 bytes (high bit set)
    ...
) };
let shuffled = unsafe { _mm256_shuffle_epi8(data, shuffle_mask) };
```

`pshufb` is the workhorse for: byte-level permutations, nibble table lookups, popcount (Wojciech Muła technique), and string operations.

### `permutevar8x32` (AVX2)

Permute 32-bit lanes within a register using a runtime index vector:

```rust
let indices = unsafe { _mm256_setr_epi32(7, 6, 5, 4, 3, 2, 1, 0) };  // Reverse order
let reversed = unsafe { _mm256_permutevar8x32_ps(data, indices) };
```

Unlike shuffle, the control comes from a register, not an immediate. This enables dynamic permutations — useful for sorting networks and filter/compact operations.

### `vperm2f128` / `vperm2i128` (AVX)

Permute 128-bit lanes between two registers:

```rust
let result = unsafe { _mm256_permute2f128_ps::<0x21>(a, b) };
// 0x21 = 00 10 00 01: low 128 = a[1], high 128 = b[0]
```

Useful for matrix transpose and interleaving data from two streams.

## Case Study: Nibble-Based Popcount

Count the number of set bits in each byte of a register using `pshufb` as a 16-entry lookup table (Wojciech Muła, 2011):

```rust
unsafe fn popcount_epi8(v: __m256i) -> __m256i {
    // Lookup table: popcount for each nibble (0-15)
    let lookup = _mm256_setr_epi8(
        0, 1, 1, 2, 1, 2, 2, 3, 1, 2, 2, 3, 2, 3, 3, 4,
        0, 1, 1, 2, 1, 2, 2, 3, 1, 2, 2, 3, 2, 3, 3, 4,
    );
    
    let low_nibble = _mm256_and_si256(v, _mm256_set1_epi8(0x0F));
    let high_nibble = _mm256_and_si256(_mm256_srli_epi16::<4>(v), _mm256_set1_epi8(0x0F));
    
    let pop_low = _mm256_shuffle_epi8(lookup, low_nibble);
    let pop_high = _mm256_shuffle_epi8(lookup, high_nibble);
    
    _mm256_add_epi8(pop_low, pop_high)
}
```

The 256-entry lookup table (one entry per byte value) would be 4×256 = 1024 bytes — doesn't fit in registers. The nibble approach uses a 16-entry table (fits in one `__m256`) and does two lookups. The `vpshufb` instruction is the table lookup: it uses each byte of `low_nibble` as an index into `lookup`.

This popcount is ~2× faster than the hardware `popcnt` instruction when counting bits in bulk (because `popcnt` has throughput 1/cycle on Zen 2, but `pshufb` can do 32 lookups in one instruction).

## Case Study: Filter / Compact

Given a vector of values and a mask, extract only the values where the mask is true into a contiguous output:

```rust
// Input:  values = [a, b, c, d, e, f, g, h]
//         mask   = [1, 0, 1, 1, 0, 1, 0, 0]
// Output: [a, c, d, f, ?, ?, ?, ?]

// Precomputed permutation table: for each 8-bit mask, a permutation that compacts matching elements
static PERMUTATION: [[u8; 8]; 256] = /* Precomputed */;

let mask_bits = unsafe { _mm256_movemask_ps(mask) };  // 8-bit mask
let perm = unsafe { _mm256_loadu_si256(PERMUTATION[mask_bits as usize].as_ptr() as *const __m256i) };
let compacted = unsafe { _mm256_permutevar8x32_ps(values, perm) };
```

The permutation table maps each possible 8-bit mask to a permutation vector. For mask `0b01011001` (bits 0, 3, 4, 6 set), the permutation is `[0, 3, 4, 6, ?, ?, ?, ?]` (the first four lanes are the set bit positions; the rest are don't-care).

Table size: 256 masks × 8 int32 = 8 KB — fits in L1. Lookup is one `_mm256_loadu_si256`. The `permutevar8x32` does the actual compaction. Total: 3 instructions to compact 8 elements. For larger compaction, repeat with 256-element blocks.

AVX-512 adds `vcompressps` which does this in hardware, eliminating the table.

## The Shuffle Strategy

When faced with a data reorganization problem in SIMD:

1. **Is there a dedicated instruction?** `vcompressps`, `vpermb`, `vpshufb`, `vpermq` — check the ISA manual.
2. **Can it be expressed with `pshufb`?** Most byte-level permutations can.
3. **Does it need a lookup table?** For dynamic permutations, precompute tables for common patterns.
4. **Can you change the data layout before the loop?** One expensive transpose before the hot loop is better than shuffling inside it.

Shuffles are cheap (~1 cycle latency, throughput 1–2/cycle) when the control is immediate. When the control comes from a register (like `permutevar`), they're slightly more expensive (~3 cycles latency). Either way, they're cheaper than memory accesses — keep data in registers and shuffle.
