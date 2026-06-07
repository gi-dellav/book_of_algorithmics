# Fast Math Approximations

Sometimes you don't need 24 bits of precision. In graphics, signal processing, and machine learning, 12–16 bits of accuracy is often sufficient, and the speedup from approximations can be 5–50×. This article covers polynomial and table-based approximations for common transcendental functions.

## The Approximation Tradeoff

| Method | Precision | Speed (relative) |
|--------|-----------|------------------|
| Hardware `expf`/`logf` (libm) | Full (24 bits) | 1× (baseline) |
| Minimax polynomial (degree 4–5) | 16–20 bits | 2–4× |
| Minimax polynomial (degree 2–3) | 10–14 bits | 4–8× |
| Table lookup + linear interp | 12–16 bits | 3–6× |
| Table lookup only (no interp) | 8–12 bits | 5–10× |
| Raw integer trick (like `rsqrt`) | 3–4 bits | 10–20× |

For a neural network inference pass where 8-bit integer weights are used, a 10-bit accurate `exp` may be indistinguishable from a full-precision one. For a physics simulation, it won't. Know your precision budget.

## Minimax Polynomials

A minimax polynomial minimizes the *maximum* error over a given interval (as opposed to Taylor series, which minimize error near zero but blow up at the edges).

The tool of choice is Sollya (https://www.sollya.org/):

```
sollya> fpminimax(exp(x), 4, [|single...|], [-0.5, 0.5]);
```

This finds the degree-4 polynomial with `float` coefficients that best approximates `exp(x)` on [−0.5, 0.5], minimizing the maximum error. Output:

```
0.9999999 + 0.9999999*x + 0.4999999*x^2 + 0.1666667*x^3 + 0.04166667*x^4
```

(These are the Taylor coefficients, because `exp` is well-approximated by its Taylor series on a small interval. For `log`, `sin`, or `tan`, the minimax coefficients differ significantly from Taylor.)

## Exp Approximation

```c
// exp(x) for x in [-0.5, 0.5], about 18 bits accurate
float exp_poly4(float x) {
    const float c0 = 0.9999999f;
    const float c1 = 0.9999999f;
    const float c2 = 0.4999999f;
    const float c3 = 0.1666667f;
    const float c4 = 0.04166667f;
    
    float x2 = x * x;
    float x4 = x2 * x2;
    // Estrin's scheme for ILP:
    return c0 + c1*x + c2*x2 + c3*x2*x + c4*x4;
}
```

For arbitrary `x`, reduce to [−0.5, 0.5] with:
```c
float expf_fast(float x) {
    // Range reduction: x = k*ln(2) + r, |r| <= ln(2)/2
    const float inv_ln2 = 1.44269504f;  // 1/ln(2)
    const float ln2 = 0.69314718f;
    
    float kf = roundf(x * inv_ln2);  // k = nearest integer to x/ln(2)
    int k = (int)kf;
    float r = x - kf * ln2;  // r in [-ln2/2, ln2/2]
    
    float exp_r = exp_poly4(r);  // Approximate exp(r)
    return ldexpf(exp_r, k);      // Multiply by 2^k (integer exponent manipulation)
}
```

The `ldexpf` call manipulates the float's exponent bits directly — it's just integer addition on the exponent field, essentially free.

## Log Approximation

```c
// log(1+x) for x in [-0.2, 0.2], about 18 bits accurate
float log1p_poly4(float x) {
    const float c1 = 1.0f;
    const float c2 = -0.4999999f;
    const float c3 = 0.3333333f;
    const float c4 = -0.2500000f;
    
    float x2 = x * x;
    float x4 = x2 * x2;
    return x * (c1 + c2*x + c3*x2 + c4*x2*x);
}
```

Full `logf_fast`:
```c
float logf_fast(float x) {
    // Extract exponent and mantissa: x = m * 2^e, m in [1, 2)
    int e;
    float m = frexpf(x, &e);  // m in [0.5, 1) or [1, 2)
    
    // Better: m in [2/3, 4/3] for faster convergence
    // ... (range reduction using sqrt(2))
    
    float log_m = log1p_poly4(m - 1.0f);
    return (float)e * 0.69314718f + log_m;
}
```

## Sin/Cos Approximation

The range reduction is the hardest part — `sin(x)` for large `x` requires high-precision `π` and careful reduction. For [−π/2, π/2]:

```c
float sin_poly5(float x) {
    const float c1 = 1.0f;
    const float c3 = -0.16666667f;  // -1/3!
    const float c5 = 0.008333333f;  //  1/5!
    const float c7 = -0.0001984127f; // -1/7!
    const float c9 = 2.75573e-6f;    //  1/9!
    
    float x2 = x * x;
    float x4 = x2 * x2;
    float x8 = x4 * x4;
    // Horner's method for sin (odd powers only):
    return x * (c1 + x2*(c3 + x2*(c5 + x2*(c7 + x2*c9))));
}
```

For `cos(x)`: use `sin_poly5(π/2 − x)` or a dedicated polynomial with even powers.

## FMA-Based Evaluation

All polynomial evaluations benefit from FMA (fused multiply-add). A Horner evaluation like `c0 + x*(c1 + x*(c2 + x*c3))` can be written:

```c
float result = fmaf(x, fmaf(x, fmaf(x, c3, c2), c1), c0);
```

Each `fmaf` does `a*b + c` with one rounding. This is both faster (3 FMA instructions vs. 3 multiplies + 3 adds) and more accurate (3 roundings instead of 6). With `-mfma`, the compiler generates FMA automatically from regular arithmetic if `-ffast-math` is enabled.

## Table-Based Methods

For irregular functions or when a polynomial would need high degree:

```c
// sin(x) via 256-entry table
const float sin_table[257];  // sin(0), sin(π/512), sin(2π/512), ..., sin(π/2)

float sin_table_lookup(float x) {
    // Assumes x in [0, π/2)
    float index_f = x * (256.0f / (M_PI/2.0f));
    int index = (int)index_f;
    float frac = index_f - index;
    // Linear interpolation
    float v0 = sin_table[index];
    float v1 = sin_table[index + 1];
    return v0 + frac * (v1 - v0);
}
```

Table size trades memory for speed+accuracy. A 256-entry table (1 KB) with linear interpolation gives ~14 bits of accuracy. A 64-entry table with cubic interpolation gives ~16 bits. A 16-entry table with no interpolation gives ~8 bits (useful for coarse approximations, e.g., graphics shaders).

## Practical Guidance

1. **Use hardware instructions first.** `rsqrtss` gives 12 bits in hardware; one Newton iteration gives 24.
2. **Use Sollya to generate minimax polynomials.** Don't use Taylor coefficients — they're suboptimal for a fixed degree.
3. **Test accuracy exhaustively.** For a single-precision function, test all 2³² inputs (takes a few minutes on a modern machine). Know the maximum error, not just the average.
4. **Document the accuracy.** "This exp approximation has maximum relative error 3×10⁻⁶ for inputs in [−1, 1]." Without this, nobody can use your code confidently.
5. **Consider `-ffast-math`.** It enables FMA, reciprocal approximations, and reassociation that can speed up math-heavy code 2–5×, at the cost of 1–2 ULPs of accuracy. For most applications, the tradeoff is worth it.
