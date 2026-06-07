# Integer Representations

Everything in a computer is bits. How those bits are interpreted as numbers is a matter of convention — conventions that run deep in the hardware and affect performance in subtle ways.

## Unsigned Integers

The simplest representation: N bits represent the values 0 through 2^N − 1.

```
8-bit unsigned: 0b10101010 = 170
```

Arithmetic is modular: it wraps around at 2^N. `255 + 1 = 0` for 8-bit unsigned. This is well-defined in C (`unsigned` overflow wraps) and directly maps to the hardware behavior of `add`.

## Signed Integers: Two's Complement

Two's complement is the universal representation for signed integers in modern hardware:

```
N-bit two's complement:
- Most significant bit has value −2^(N−1) instead of +2^(N−1).
- Range: [−2^(N−1), 2^(N−1) − 1]
- 8-bit: −128 to 127
```

Key properties:
- Zero has exactly one representation (all zeros).
- Negation: `~x + 1` (flip all bits and add 1).
- Addition and subtraction use the exact same hardware as unsigned.
- The most negative number is its own negative: `-128` negated is still `-128` in 8-bit (overflow).

Why two's complement won: it's the only representation where addition and subtraction of signed numbers is identical to unsigned at the bit level. No special signed-arithmetic hardware needed.

## Endianness

How multi-byte integers are stored in memory:

**Little-endian** (x86, ARM default): The least significant byte is stored at the lowest address.
```
0x12345678 stored at address 0:
  [0x78] [0x56] [0x34] [0x12]
  0      1      2      3
```

**Big-endian** (some ARM, network byte order): The most significant byte at the lowest address.
```
0x12345678 stored at address 0:
  [0x12] [0x34] [0x56] [0x78]
  0      1      2      3
```

x86 is natively little-endian. ARM can be configured for either (AArch64 is mostly little-endian in practice). Network protocols use big-endian (hence `htonl`/`ntohl`).

Endianness matters when:
- Sending data over the network.
- Reading binary file formats.
- Type-punning (accessing the same bytes as different types).
- SIMD shuffles (byte ordering within vectors).

Most of the time, you don't need to care. When you do, `<endian.h>` (C++20: `<bit>` header with `std::endian`) provides compile-time detection and conversion.

## 128-bit Integers

x86-64 has partial support for 128-bit integers through the `mul` instruction. Multiplying two 64-bit numbers produces a 128-bit result: the low 64 bits in `rax`, the high 64 bits in `rdx`.

```c
// GCC/Clang extension:
__int128 a = 0x123456789ABCDEF0;
__int128 b = 0x0FEDCBA987654321;
__int128 product = a * b;  // Uses mul/imul
```

The compiler handles `__int128` arithmetic with library calls for operations beyond multiplication (division, modulo, shift by variable amount). 128-bit add/sub are cheap (two-instruction sequences). 128-bit division is expensive (software implementation).

## Integer Overflow and UB

C and C++ diverge sharply here:

- **Unsigned overflow**: Defined behavior. Wraps modulo 2^N.
- **Signed overflow**: Undefined behavior. The compiler may assume it never happens.

```c
int a = INT_MAX;
int b = a + 1;  // UB! Compiler may optimize this away or generate unexpected code.

unsigned c = UINT_MAX;
unsigned d = c + 1;  // Defined, wraps to 0.
```

The UB of signed overflow enables important optimizations. See `compilation/contracts.md` for details. For safe signed arithmetic, use compiler builtins:

```c
int result;
if (__builtin_add_overflow(a, b, &result)) {
    // Handle overflow
}
// Or use: __builtin_sadd_overflow, __builtin_ssubl_overflow, __builtin_smul_overflow
```

These generate efficient code (a single `add` + `jo` / `jno` sequence) rather than expensive checking logic.

## Signed Right Shift

C leaves the right shift of signed negative integers **implementation-defined** (not UB, but not portable):

```c
int x = -8;
int y = x >> 1;  // Implementation-defined: -4 on most platforms (arithmetic shift)
```

x86's `sar` (Shift Arithmetic Right) preserves the sign bit, filling with 1s for negative numbers. Most compilers use `sar` for signed `>>`. But if portability matters, use unsigned types for bit manipulation.
