# Fast Inverse Square Root

In 1999, id Software released Quake III Arena. Buried in the source code was this:

```rust
fn q_rsqrt(number: f32) -> f32 {
    let x2: f32;
    let y: f32;
    let i: i32;
    let threehalfs: f32 = 1.5;

    x2 = number * 0.5;
    y  = number;
    i  = unsafe { std::mem::transmute::<f32, i32>(y) };  // Evil floating-point bit hack
    i  = 0x5f3759df - (i >> 1);  // What the fuck?
    y  = unsafe { std::mem::transmute::<i32, f32>(i) };
    y  = y * (threehalfs - (x2 * y * y));  // 1st iteration
    // y  = y * (threehalfs - (x2 * y * y));  // 2nd iteration (removed)

    y
}
```

This function computes `1/√x` — the inverse square root — about 4× faster than `1.0f / sqrtf(x)` on 1999 hardware, with an error of less than 0.2%. It became the most famous piece of numerical code ever written. Here's how it works.

## The Problem

Computing `1/√x` is expensive:
- Hardware `sqrt` takes ~20 cycles (Pentium III era).
- Hardware division takes ~20 cycles.
- `1.0 / sqrt(x)` does both — ~40 cycles.

For a game engine that needs to normalize millions of vectors per frame, this is a bottleneck. Normalizing a vector `(x, y, z)` requires dividing each component by `sqrt(x² + y² + z²)` — one square root and three divisions per vector.

## The Insight

Take the logarithm of both sides:

```
y = 1/√x
log₂(y) = −½ × log₂(x)
```

Now, an IEEE 754 float stored as bits and interpreted as an integer gives a piecewise-linear approximation of `log₂(x)`:

```
float x:   [s][eeeeeeee][mmmmmmmmmmmmmmmmmmmmmmm]
            ^  exponent   mantissa

As integer: ≈ 2^23 × (E − 127) + M     (where E = exponent, M = mantissa)
           = 2^23 × (E + M/2^23 − 127)
           = 2^23 × (log₂(x) + 127 − correction)
```

The "correction" term accounts for the fact that `log₂(1 + M/2^23)` is approximately `M/2^23` but not exactly (it's slightly curved, and there's an offset). Empirically, the best constant is around `σ ≈ 0.04504656`.

So:
```
bits(x) ≈ 2^23 × (log₂(x) + 127 − σ)
```

Therefore:
```
log₂(x) ≈ bits(x) / 2^23 − 127 + σ
```

## The Magic

We want `y = 1/√x`, so `log₂(y) = −½ × log₂(x)`.

Substituting the approximation:

```
bits(y)/2^23 − 127 + σ ≈ −½ × (bits(x)/2^23 − 127 + σ)

bits(y) ≈ 3/2 × 2^23 × (127 − σ) − bits(x)/2
        = 0x5F3759DF − (bits(x) >> 1)
```

The magic constant `0x5F3759DF` is `3/2 × 2^23 × (127 − σ)` with σ tuned to minimize the maximum error over the range (0, ∞). The value was found experimentally — Chris Lomont later derived a slightly better constant (`0x5F375A86`) analytically, but the original is within 0.001% of optimal.

## Newton Refinement

The bit hack gives an initial guess with about 3% relative error. One iteration of Newton's method for `f(y) = 1/y² − x`:

```
y_{n+1} = y × (1.5 − 0.5 × x × y × y)
```

reduces the error to ~0.2%. The code uses the equivalent but slightly faster form:
```rust
y = y * (threehalfs - (x2 * y * y));  // x2 = number * 0.5
```

A second iteration would reduce error to ~0.000001% but wasn't needed for Quake's purposes (rendering at 640×480).

## Modern Relevance

On modern hardware, `Q_rsqrt` is **not faster** than `1.0f / sqrtf(x)`:
- Hardware `sqrtps` (SIMD) has latency ~13 and throughput ~6 on Zen 2.
- Hardware division throughput is ~1/5 per cycle for scalar, better for SIMD.
- `rsqrtps` (SSE/AVX) gives a 12-bit accurate `1/√x` in hardware, which can be refined with Newton.

But the technique — integer reinterpretation of floats for function approximation — remains powerful:
- Initialization for iterative solvers (Newton, gradient descent).
- Low-precision SIMD approximations where throughput matters more than accuracy.
- Embedded systems without FPUs.
- Understanding the deep connection between IEEE 754 encoding and the logarithm function.

The `rsqrt` story is also a lesson in the evolution of hardware: what was a brilliant optimization in 1999 became a standard instruction (RSQRTSS), then was supplanted by general-purpose throughput improvements (faster `sqrt` + `div` + FMA). Today's clever hack is tomorrow's ISA extension.
