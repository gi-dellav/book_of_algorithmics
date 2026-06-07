# Floating-Point Numbers in Depth

This article builds intuition for floating-point arithmetic by constructing a floating-point system from scratch. The goal is not to teach IEEE 754 (that's the previous article) but to develop a feel for *why* floating-point works the way it does.

## Fixed-Point: The Starting Point

Before floats, there were fixed-point numbers. A 32-bit integer where we decree that the binary point is at a fixed position:

```
32-bit fixed-point with 16 fractional bits:
  [ integer part (16 bits) ] [ fractional part (16 bits) ]
  
  Value = integer_part + fractional_part / 2^16
  Range: [-32768, 32767.99998]
  Precision: 1/65536 ≈ 0.000015
```

Fixed-point has constant absolute precision. Every representable number is equally spaced. This is great for applications with a known, bounded range (audio samples, pixel coordinates). It's terrible for general computing — you can't represent both the mass of the sun and the mass of an electron with the same fixed-point format.

## Floating-Point: Making the Point Float

The insight: instead of a fixed binary point, store the point's position alongside the digits. This is scientific notation in binary.

```
x = sign × significand × 2^exponent
```

In a 32-bit word, we might allocate:
- 1 bit for sign.
- 8 bits for exponent (signed integer, stored with a bias).
- 23 bits for significand (the fractional part; the leading 1 is implicit).

This gives **relative** precision: the spacing between numbers is proportional to their magnitude. Near 1.0, numbers are spaced by 2^−23 ≈ 1.2×10^−7. Near 10^38, they're spaced by 10^31. The relative error of any representation is bounded by 2^−24 ≈ 6×10^−8.

That's the fundamental tradeoff: fixed-point has constant absolute error; floating-point has constant relative error. For most computations, relative error is what matters.

## Building a DIY Float

Let's construct a tiny floating-point system: 1 sign bit, 3 exponent bits (bias 3), 2 mantissa bits.

```
Bits: [s][eee][mm]

Values (positive side):
 0 000 00 = 0
 0 000 01 = 0.15625  (denormalized)
 0 000 10 = 0.3125   (denormalized)
 0 000 11 = 0.46875  (denormalized)
 0 001 00 = 0.5      (normalized, exponent=−2)
 0 001 01 = 0.625
 ...
 0 110 11 = 14.0     (largest normalized)
 0 111 00 = +inf
 0 111 01 = NaN
```

Key observations:
- **The spacing doubles at each power of two.** Between 1.0 and 2.0, numbers are spaced by 0.25. Between 2.0 and 4.0, by 0.5. Between 0.5 and 1.0, by 0.125.
- **Subnormals fill the gap between 0 and the smallest normal.** Without them, there would be a sudden gap from ±0 to ±0.5. Subnormals provide gradual underflow, at the cost of performance and precision (the leading 1 is gone).
- **The number line is not uniformly dense.** Half of all representable floats lie between −1 and 1. A quarter lie between −0.5 and 0.5. The representable values cluster around zero, where we need them most.

## The IQ Bell Curve

There's a delightful observation: the bit pattern of a float, interpreted as a two's complement integer, is monotonic for positive numbers. That is, if `a > b > 0` as floats, then `*(int*)&a > *(int*)&b` as integers. This is deliberate in IEEE 754 — it means that integer comparison hardware can compare positive floats without modification.

This property is what makes the fast inverse square root possible. By interpreting a float's bits as an integer, we can manipulate it with integer operations (shifts, subtraction) that approximate complex floating-point functions. See `rsqrt.md` for the full derivation.

## Hardware Floating-Point

Modern CPUs implement floating-point in dedicated execution units:
- **x87** (legacy): 80-bit extended precision stack-based FPU. Slow, stack-based, largely obsolete. Avoid `long double` on x86 if you care about performance.
- **SSE/AVX scalar**: 32-bit and 64-bit operations on `xmm` registers. This is what modern compilers use for `float` and `double`.
- **SIMD packed**: Same operations on vectors. FMA, masking, blending.

The hardware supports:
- `add`, `sub`, `mul`, `div`, `sqrt` (all IEEE 754 compliant).
- `fma` (fused multiply-add, 1 rounding instead of 2).
- Conversions between formats (float→int, int→float, float→double, etc.).
- Comparisons (producing masks for SIMD, or FLAGS for scalar).

All at latencies of 3–5 cycles for add/mul/FMA (Zen 2). Division and sqrt are slower (13–44 cycles).

## Common Pitfalls

1. **`0.1 + 0.2 != 0.3`**: These decimal fractions are repeating fractions in binary, just as 1/3 is in decimal. Rounding to 24 or 53 bits introduces tiny errors. Use epsilon comparisons, not equality.
2. **Catastrophic cancellation**: Subtracting nearly-equal numbers loses significant digits. `(1.0000001 − 1.0) = 1.0e−7` — only one significant digit remains. Restructure computations to avoid.
3. **Float-to-int conversion is slow**: Converting between float and int domains requires changing the rounding mode and moving data between FP and integer register files. Minimize in hot code.
4. **Denormals kill performance**: A few denormal numbers in a data stream can slow the entire computation to a crawl. Normalize your data or enable flush-to-zero.
