# Rounding Errors

Floating-point arithmetic is approximate. Every operation rounds the exact mathematical result to the nearest representable value. For a single operation, the error is tiny (~10^−7 for float, ~10^−16 for double). For millions of operations, errors can accumulate, cancel, or amplify. This article covers the theory and practice of managing rounding errors.

## Machine Epsilon

Machine epsilon (ε) is the difference between 1.0 and the next representable float:

- `float`: ε ≈ 1.19×10^−7 (2^−23)
- `double`: ε ≈ 2.22×10^−16 (2^−52)

Every floating-point operation introduces a relative error of at most ε/2 (with the default rounding mode). This is the "unit in the last place" (ULP) — one ULP is ε × |value|.

```rust
let x: f32 = 16777216.0;  // Exactly representable
let y: f32 = x + 1.0;     // y == 16777216.0! (1.0 is less than ε × x)
```

When |x| > 1/ε ≈ 1.68×10^7, consecutive floats are spaced more than 1 apart. Adding 1 does nothing. This trips up loop counters: `for (float x = 0; x < 1e8; x++)` may be infinite because at some point `x += 1` stops changing `x`.

## Sources of Error

### Rounding Error (every operation)

```rust
let a: f32 = 1.0 / 3.0;  // 0.33333334 (not exactly 1/3)
```

Each arithmetic operation rounds its result to the nearest representable float. The error is random (in sign) and bounded by ε/2.

### Catastrophic Cancellation

```rust
let a: f32 = 1.0000001;
let b: f32 = 1.0000000;
let diff: f32 = a - b;  // 1.0e-7 (only ~1 significant digit!)
```

When two nearly-equal numbers are subtracted, the most significant digits cancel out, leaving only the low-order bits — which are mostly rounding error from previous operations. The relative error in `diff` can be enormous.

Fix: restructure the computation. For `sqrt(x+1) − sqrt(x)`, use the identity:
```
sqrt(x+1) − sqrt(x) = 1 / (sqrt(x+1) + sqrt(x))
```
No cancellation, much lower error.

### Accumulation Error

```rust
let mut sum: f32 = 0.0;
for i in 0..n {
    sum += a[i];  // Error grows with n
}
```

Each addition introduces ~ε/2 relative error. After n additions, the error is O(√n × ε) for random data (errors partially cancel), or O(n × ε) for worst-case data (errors are correlated).

The absolute error when summing n numbers of similar magnitude is roughly n × ε × |average|. For n = 10⁶ and float, the error can be ~0.1 × |average| — the sum is garbage.

## Kahan Summation

Kahan summation (compensated summation) reduces the error to O(ε) regardless of n:

```rust
let mut sum: f32 = 0.0;
let mut compensation: f32 = 0.0;  // Running lost low-order bits

for i in 0..n {
    let y = a[i] - compensation;  // Apply correction from previous step
    let t = sum + y;               // New sum
    compensation = (t - sum) - y;    // Recover the low bits lost in the addition
    sum = t;
}
```

The trick: `(t - sum)` recovers the high-order bits of `y` that were actually added. Subtracting `y` from that gives the *low-order* bits of `y` that were lost (with reversed sign). These are carried to the next iteration.

For n = 10⁶, Kahan summation with `float` gives accuracy comparable to naive summation with `double`. Cost: 4× more operations per element. For most applications, just use `double` — it's simpler and gives you 2^29× more precision.

## Double-Double Arithmetic

For when `double` isn't enough: represent a number as the sum of two doubles (a high part and a low part). The high part holds the most significant ~53 bits; the low part holds the next ~53 bits. Total precision: ~106 bits (~32 decimal digits).

```rust
// Double-double addition (Dekker's algorithm)
fn dd_add(a_hi: f64, a_lo: f64, b_hi: f64, b_lo: f64, r_hi: &mut f64, r_lo: &mut f64) {
    let s = a_hi + b_hi;
    let t = if s - a_hi > b_hi { a_hi - (s - b_hi) } else { b_hi - (s - a_hi) };
    *r_hi = s;
    *r_lo = (a_lo + b_lo) + t;
}
```

Double-double is used in high-precision math libraries and for implementing correctly-rounded transcendental functions. It's ~5–10× slower than `double` but ~1,000× faster than arbitrary-precision software libraries.

## Interval Arithmetic

Instead of storing a single approximate value, store an interval [lower, upper] that is guaranteed to contain the exact result:

```rust
struct Interval { lo: f32, hi: f32 }

fn add(a: Interval, b: Interval) -> Interval {
    Interval { lo: a.lo + b.lo, hi: a.hi + b.hi }  // With appropriate rounding
}
```

For reliable bounds, you must control rounding direction: `a.lo + b.lo` rounded toward −∞, `a.hi + b.hi` rounded toward +∞. This requires changing the FPU's rounding mode (expensive) or using software rounding.

Interval arithmetic is used in formal verification and some scientific computing applications. For most performance work, it's overkill — use double and check against known results.

## Practical Recommendations

1. **Prefer `double` over `float` unless memory bandwidth is the bottleneck.** Double's 53-bit mantissa gives enormous headroom for accumulation error.
2. **Sort numbers before summing**, or use pairwise summation (recursively sum halves). Reduces error from O(n) to O(log n).
3. **Avoid subtracting nearly-equal numbers.** Restructure the math.
4. **Use `-ffast-math` only when the extra 1–2 ULPs of error don't matter.** For most ML, graphics, and signal processing, it doesn't.
5. **Test your numerics.** Compare double-precision and single-precision results. Compare with a known reference implementation. Plot the error distribution.
