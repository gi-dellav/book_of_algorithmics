# Newton's Method

Newton's method (Newton-Raphson) is the workhorse of numerical computing. It finds roots of functions with quadratic convergence — each iteration doubles the number of correct digits. It's used for computing square roots, reciprocals, and transcendental functions, both in software and in the hardware itself.

## The Method

Given a differentiable function `f(x)` and an initial guess `x₀` for a root (where `f(x) = 0`):

```
x_{n+1} = x_n − f(x_n) / f'(x_n)
```

Geometrically: draw the tangent line at `(x_n, f(x_n))`. The next guess is where that tangent crosses the x-axis.

For finding √a:

Set `f(x) = x² − a`. The root is `√a`. Then:
- `f'(x) = 2x`
- `x_{n+1} = x − (x² − a) / (2x) = (x + a/x) / 2`

```c
float sqrt_newton(float a) {
    float x = a / 2.0f;  // Initial guess
    for (int i = 0; i < 4; i++) {
        x = (x + a / x) / 2.0f;
    }
    return x;
}
```

Four iterations give ~16 correct digits (for float, 3 iterations is usually enough).

## Quadratic Convergence

Let εₙ = |xₙ − x*| be the error after iteration n. Newton's method satisfies:

```
ε_{n+1} ≈ (εₙ)² × |f''(x*)| / (2 × |f'(x*)|)
```

Each iteration *squares* the error! If ε₀ = 0.1, then:
- ε₁ ≈ 0.01 (2 correct digits)
- ε₂ ≈ 0.0001 (4 correct digits)
- ε₃ ≈ 10⁻⁸ (8 correct digits)
- ε₄ ≈ 10⁻¹⁶ (16 correct digits — double precision saturated)

This is why Newton's method converges in 3–5 iterations for most functions and reasonable initial guesses.

## Case Study: Computing 1/√x

To compute the inverse square root without a division or square root in hardware:

Goal: find `y = 1/√a` (so that `y² = 1/a`).

Set `f(y) = 1/y² − a`. The root is `1/√a`. Then:
- `f'(y) = −2/y³`
- `y_{n+1} = y − (1/y² − a) / (−2/y³) = y × (3 − a×y²) / 2`

```c
float rsqrt_newton(float a) {
    float y = 1.0f / sqrtf(a);  // Initial guess (uses hardware sqrt)
    y = y * (3.0f - a * y * y) * 0.5f;  // One Newton iteration
    y = y * (3.0f - a * y * y) * 0.5f;  // Second iteration
    return y;
}
```

Each iteration uses only multiplication and addition/subtraction — no division, no square root. This is how the Quake III `rsqrt` works; the next article covers how they got the initial guess without hardware `sqrt`.

## Hardware Utilization of Newton's Method

The CPU's own division and square root units use iterative algorithms (SRT division, Goldschmidt's algorithm for division, Newton-Raphson for square root). The hardware implements these with radix-4 or radix-8 iterations, producing 2–3 bits per cycle. A 64-bit division requires ~20 cycles because the hardware needs ~20 iterations to get 64 correct bits.

Software Newton iteration is sometimes faster than hardware for specific cases:
- When you need low precision (3–4 correct digits, not 15).
- When you can amortize the cost over many values (one initial guess, many refinements).
- When the hardware doesn't have the operation at all (e.g., `1/√x` on old hardware).

## Initial Guesses

Newton's method is only as good as the starting point. Common techniques:

- **Table lookup**: For √x, use the exponent bits to get a rough estimate, then look up the mantissa in a small table (128–256 entries). Accuracy: ~7 bits. One Newton iteration gives ~14 bits; two give ~28.
- **Polynomial approximation**: Fit a low-degree polynomial to the function over a small interval. Two to four terms give 8–12 bits of accuracy.
- **Hardware instructions**: `rsqrtss`/`rsqrtps` (SSE) provide a 12-bit accurate initial guess for `1/√x` in hardware. One Newton iteration gives full single-precision accuracy.
- **Integer reinterpretation**: For `1/√x` and `√x`, treating the float as an integer and subtracting from a magic constant gives a surprisingly good initial guess. (This is the Quake III trick — see `rsqrt.md`.)

## Beyond Square Roots

Newton's method generalizes to:
- **Any root**: `x_{n+1} = ((k-1)x_n + a/x_n^{k-1}) / k` for the k-th root of a.
- **System of equations**: The Jacobian replaces the derivative. Each step solves a linear system.
- **Optimization**: Newton's method on the gradient finds minima/maxima (requires the Hessian matrix).
- **Division**: `x_{n+1} = x_n × (2 − d × x_n)` iteratively computes `1/d`. Two iterations with good initial guess give 32-bit precision.

For most performance work, you won't implement Newton's method from scratch — you'll use hardware instructions or library functions. But understanding it explains *why* those instructions work, and when a customized implementation might be faster (lower precision, amortized initial guess, SIMD).
