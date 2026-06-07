# IEEE 754 Floating-Point

IEEE 754 (1985, revised 2008 and 2019) is the standard that defines how floating-point numbers work on essentially all modern hardware. It specifies formats, rounding modes, exception handling, and special values. Understanding it is not optional for anyone doing numerical or performance work.

## Formats

| Format | Bits | Sign | Exponent | Mantissa | C type |
|--------|------|------|----------|----------|--------|
| Half (binary16) | 16 | 1 | 5 | 10 | `_Float16` (C23) |
| Single (binary32) | 32 | 1 | 8 | 23 | `float` |
| Double (binary64) | 64 | 1 | 11 | 52 | `double` |
| Extended (x87 80-bit) | 80 | 1 | 15 | 64 | `long double` (x86) |
| Quad (binary128) | 128 | 1 | 15 | 112 | `_Float128` (GCC) |
| bfloat16 | 16 | 1 | 8 | 7 | `__bf16` (GCC) |

All formats use the same structure: `(−1)^sign × 2^(exponent−bias) × (1 + mantissa/2^M)`.

For single precision:
- Bias = 127.
- Value = `(−1)^s × 2^(e−127) × (1.m)` where `m` is the 23-bit mantissa.
- The leading 1 is implicit (not stored) — it's the "hidden bit."

## Special Values

| Exponent | Mantissa | Value |
|----------|----------|-------|
| All 0 | All 0 | ±0 |
| All 0 | Nonzero | Denormalized (subnormal) |
| All 1 | All 0 | ±∞ |
| All 1 | Nonzero | NaN (Not a Number) |

**Subnormal numbers**: When the exponent field is all zeros, the implicit leading bit becomes 0 instead of 1. This allows representing very small numbers (down to ~1.4×10^−45 for single precision) but with reduced precision. Subnormals are computed using microcode on many CPUs and are **extremely slow** — 100+ cycles. Flushing them to zero (`-ffast-math` or setting FTZ/DAZ flags) can dramatically speed up code that accidentally creates subnormals.

**NaN**: Represents "not a number" — the result of `0.0/0.0`, `sqrt(−1)`, or `inf − inf`. NaN ≠ NaN (always false, even for the same NaN). There are signaling NaNs (which raise exceptions) and quiet NaNs (which propagate silently). Most libraries use quiet NaNs for error propagation.

**Signed zeros**: +0 and −0 are distinct but compare equal. `1.0/+0.0 = +∞`; `1.0/−0.0 = −∞`. Signed zeros preserve sign in expressions that underflow.

## Encoding

Single precision layout:
```
bit 31    bits 30-23    bits 22-0
[ sign ] [ exponent ] [ mantissa ]
```

Example: `−118.625`
1. Sign: negative → `1`.
2. 118.625 in binary: `1110110.101` (118 = 1110110, 0.625 = 0.101).
3. Normalize: `1.110110101 × 2^6`.
4. Exponent: 6 + 127 (bias) = 133 = `10000101`.
5. Mantissa (fractional part only): `11011010100000000000000`.
6. Result: `1 10000101 11011010100000000000000`.

## Rounding Modes

IEEE 754 specifies five rounding modes:

- **Round to nearest, ties to even** (default): The mathematically closest value; if exactly halfway, choose the one with an even least significant bit.
- **Round toward +∞** (ceiling).
- **Round toward −∞** (floor).
- **Round toward 0** (truncation).

Rounding mode affects every floating-point operation. The default (ties to even) minimizes accumulated rounding bias over many operations. Changing the rounding mode affects performance — non-default modes may prevent certain optimizations.

## Why IEEE 754 Matters for Performance

1. **Gradual underflow (subnormals)**: Slow on most hardware. Flush to zero for performance if your algorithm can tolerate it.
2. **NaN propagation**: Compilers can't reorder floating-point operations across NaN checks without `-ffast-math`.
3. **Rounding mode**: The default (ties to even) is what FMA, SIMD, and most hardware is optimized for. Changing it disables many optimizations.
4. **FMA**: `(a * b) + c` with a single rounding step. IEEE 754-2008 specifies FMA, and hardware (except some GPUs) follows it. More accurate *and* faster.
5. **Reproducibility**: IEEE 754 guarantees bit-exact results across implementations for basic operations. But optimizations like FMA contraction and `-ffast-math` break reproducibility. Know when you need it.

## bfloat16

Google's bfloat16 ("brain floating point") is a 16-bit format with the same exponent range as float32 (8 exponent bits) but only 7 mantissa bits. It's designed for ML workloads where dynamic range matters more than precision. Conversion to/from float32 is a simple truncation/zero-extension of the mantissa. Supported in recent CPUs (Intel Cooper Lake+, AMD Zen 4+, ARM AArch64 with BF16).

Key advantage: training neural networks in bfloat16 can be nearly as accurate as float32 while using half the memory bandwidth.
