# Bit Hacks

Some operations are faster with bit manipulation than with arithmetic. This article catalogs the bit hacks that remain relevant — the ones compilers don't already do for you, or that are useful in SIMD and SWAR contexts.

## SWAR: SIMD Within A Register

Before SIMD instructions, clever programmers packed multiple values into a single register and used bitwise operations to process them in parallel. This technique is called SWAR, and it's still useful today — especially for operations on short strings, byte arrays, and packed data.

**Example: Detect if any byte in a 64-bit word is zero:**

```c
uint64_t has_zero_byte(uint64_t x) {
    return (x - 0x0101010101010101ull) & ~x & 0x8080808080808080ull;
}
```

How it works:
- `x - 0x0101010101010101`: For each byte, subtract 1. If a byte is 0, borrowing from the next byte sets the high bit of the current byte.
- `~x`: Clear bits where the original byte had the high bit set (avoids false positives for bytes ≥ 128).
- `& 0x8080808080808080`: Extract the high bit of each byte.

This is how `strlen` and `strchr` can scan 8 bytes at a time without SIMD. It's also how `memchr` works on architectures without SIMD.

**Example: Population count (SWAR):**

```c
uint64_t popcnt_swar(uint64_t x) {
    x = (x & 0x5555555555555555ull) + ((x >> 1) & 0x5555555555555555ull);
    x = (x & 0x3333333333333333ull) + ((x >> 2) & 0x3333333333333333ull);
    x = (x & 0x0F0F0F0F0F0F0F0Full) + ((x >> 4) & 0x0F0F0F0F0F0F0F0Full);
    x = (x * 0x0101010101010101ull) >> 56;
    return x;
}
```

On modern hardware, use `__builtin_popcountll` — it compiles to the `popcnt` instruction (throughput 4/cycle on Zen 2). But the SWAR version illustrates the technique: sum bits in groups of 2, then 4, then 8, then multiply-and-shift to sum all bytes.

## Bit Reversal

Reversing the bits of a word (bit 0 becomes bit 63, bit 1 becomes bit 62, etc.):

```c
uint64_t reverse_bits(uint64_t x) {
    x = ((x & 0x5555555555555555ull) << 1) | ((x >> 1) & 0x5555555555555555ull);
    x = ((x & 0x3333333333333333ull) << 2) | ((x >> 2) & 0x3333333333333333ull);
    x = ((x & 0x0F0F0F0F0F0F0F0Full) << 4) | ((x >> 4) & 0x0F0F0F0F0F0F0F0Full);
    x = __builtin_bswap64(x);  // Reverse bytes (hardware instruction)
    return x;
}
```

Bit reversal is needed for FFT bit-reversal permutation and some CRC algorithms. The x86 `bswap` instruction reverses bytes; the nibble-swapping is done in software.

## Power-of-Two Rounding

Round up to the next power of two:

```c
uint64_t next_pow2(uint64_t x) {
    x--;
    x |= x >> 1;
    x |= x >> 2;
    x |= x >> 4;
    x |= x >> 8;
    x |= x >> 16;
    x |= x >> 32;
    return x + 1;
}
```

This "smears" the highest set bit across all lower bits, then adds 1. Branchless, constant time. Used everywhere in memory allocators and hash table implementations.

## Least Significant Bit

Isolate the least significant set bit:

```c
x & -x  // LSB isolation (two's complement: -x = ~x + 1)
```

Example: `x = 0b101100`, `x & -x = 0b000100`. Used in Fenwick trees (binary indexed trees), sparse bitsets, and the `blsi` instruction.

Clear the LSB:
```c
x & (x - 1)  // Clears lowest set bit
```

Used to iterate over set bits:
```c
while (x) {
    uint64_t lsb = x & -x;
    process(ctz(lsb));  // ctz = count trailing zeros (tzcnt instruction)
    x &= x - 1;
}
```

## Bitfield Extract/Insert

Modern x86 has `bextr` and `bzhi` (BMI1/BMI2) for fast bitfield operations:

```c
// Extract bits [start, start+len) from x
uint64_t result = _bextr_u64(x, start, len);

// Zero high bits starting at position
uint64_t result = _bzhi_u64(x, position);
```

These are single-cycle operations. Use the intrinsics directly when manipulating packed bitfields — much faster than shift-mask sequences.

## Population Count Applications

`popcnt` is not just for bit counting:

- **Hamming distance**: `popcnt(a ^ b)` — number of bits that differ between two values.
- **Sparse bitset size**: `popcnt(bitmap)` — number of elements in the set.
- **SIMD comparison counting**: After `_mm256_cmp_ps`, use `popcnt` on the movemask result to count how many elements matched.
- **Parity**: `popcnt(x) & 1` — even or odd population.

On Zen 2, `popcnt` has throughput 4/cycle — it's as cheap as integer addition.

## When NOT to Use Bit Hacks

Most bit hacks from Sean Anderson's "Bit Twiddling Hacks" (2005) are obsolete:
- Compilers now recognize common patterns and emit the optimal instruction.
- Hardware instructions (`popcnt`, `lzcnt`, `tzcnt`, `blsi`, `blsmsk`, `bzhi`) directly implement the most common operations.
- Readability matters. `x & -x` is idiomatic and every systems programmer knows it. `((x ^ (x >> 31)) - (x >> 31))` for absolute value is not — use `abs(x)` and trust the compiler.

**Rule**: check the assembly. If your clever hack compiles to more instructions than the obvious code, it's not clever — it's harmful.
