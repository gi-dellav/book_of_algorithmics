# Random Sampling and Monte Carlo

Every statistical simulation starts with random numbers. The quality and speed of your random number generator determine the quality and speed of everything downstream. The algorithms in this article generate 10⁹ uniform variates per second and transform them into any distribution you need.

## Uniform Random Numbers

Chapter 7 (Number Theory) covered RNG algorithms: xorshift, PCG, and the statistical tests they must pass. Here we focus on throughput. On Zen 2, a single xorshift128+ instance generates ~2.5 billion `u64` values per second (~3 cycles each). But statistical simulation needs `f64` in [0, 1). The conversion matters:

```rust
// Fast: use only the upper 53 bits (mantissa of f64)
fn u64_to_f64(x: u64) -> f64 {
    (x >> 11) as f64 * 2.0f64.powi(-53)
}

// Faster: bitwise OR with the exponent pattern for [1.0, 2.0), then subtract 1.0
fn u64_to_f64_fast(x: u64) -> f64 {
    let bits = (x >> 12) | 0x3FF0000000000000u64;
    f64::from_bits(bits) - 1.0
}
```

The bitwise method avoids integer-to-float conversion entirely. It generates a float in [1.0, 2.0) by setting the exponent to 1023 (bias) and filling the mantissa with random bits, then subtracts 1.0. This is ~5× faster than `(x as f64) / (u64::MAX as f64)` because it avoids the expensive `cvtsi2sd` (integer to float) instruction.

For SIMD generation of 8 floats at a time:

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

unsafe fn rand_uniform_avx2(rng: &mut XorShift128, output: &mut [f64]) {
    for chunk in output.chunks_mut(8) {
        let vals = [
            (rng.next() >> 12) | 0x3FF0000000000000u64,
            (rng.next() >> 12) | 0x3FF0000000000000u64,
            // ... 6 more
        ];
        let ones = _mm256_set1_pd(1.0);
        let raw = _mm256_loadu_pd(vals.as_ptr() as *const f64);
        let result = _mm256_sub_pd(raw, ones);
        _mm256_storeu_pd(chunk.as_mut_ptr(), result);
    }
}
```

~5 billion doubles/second — memory bandwidth starts to become the bottleneck.

## Inverse Transform Sampling

Given a CDF F(x) (monotone, 0→1), generate uniform u ~ Unif(0,1), return F⁻¹(u). For distributions with a closed-form inverse CDF:

```rust
fn sample_exponential(lambda: f64, u: f64) -> f64 {
    -u.ln() / lambda  // F⁻¹(u) = -ln(1-u)/λ, and 1-u ~ Unif(0,1)
}
```

The bottleneck is `ln()` — ~15 cycles on Zen 2. For 10⁹ samples: ~0.6 seconds. Acceptable.

## Box-Muller for Normal Distribution

Transform two uniforms into two independent normals:

```rust
fn box_muller(u1: f64, u2: f64) -> (f64, f64) {
    let r = (-2.0 * u1.ln()).sqrt();
    let theta = 2.0 * std::f64::consts::PI * u2;
    (r * theta.cos(), r * theta.sin())
}
```

Two `ln()`, one `sqrt`, one `sin`, one `cos` per two samples. ~60 cycles per pair (~30 cycles per sample). For 10⁸ samples: ~1.0 seconds.

## Ziggurat Algorithm

The Ziggurat is 3–4× faster than Box-Muller by avoiding transcendental functions ~98% of the time. It precomputes a table of rectangles covering the normal PDF, samples uniformly from the rectangles, and occasionally (2% of the time) falls back to a tail-sampling routine:

```rust
struct Ziggurat {
    x: [f64; 256],   // right edges of rectangles
    y: [f64; 256],   // heights of rectangles
}

impl Ziggurat {
    fn sample_normal(&self, rng: &mut impl Rng) -> f64 {
        loop {
            // Randomly pick a rectangle
            let i = (rng.gen::<u64>() & 0xFF) as usize;

            // Sample x uniformly in [-x[i], x[i]]
            let u: f64 = rng.gen_range(-1.0..1.0);
            let x = u * self.x[i];

            // Fast check: is (x, u*y[i]) under the PDF?
            if x.abs() < self.x[i + 1] {
                return x;  // 98.5% of cases — no transcendental functions
            }

            // Slow path: rectangular wedge test + tail sampling
            if i == 0 {
                return sample_tail(rng); // ~0.3% of cases
            }
            let y = rng.gen_range(0.0..self.y[i]);
            if y * self.pdf(x) < 1.0 {
                return x; // ~1.2% of cases
            }
        }
    }
}
```

~8 cycles per sample (98.5% hit rate). ~4 billion normal samples/second on Zen 2. The Ziggurat is the default in NumPy, Julia, and R.

## Rejection Sampling

When the inverse CDF is unknown and the Ziggurat doesn't apply, use rejection sampling. Sample from a proposal distribution g(x) that envelopes the target f(x), and accept with probability f(x) / (M · g(x)):

```rust
fn rejection_sample<F, G>(target: F, proposal: G, m: f64,
                           rng: &mut impl Rng) -> f64
where F: Fn(f64) -> f64, G: Fn(&mut impl Rng) -> f64 {
    loop {
        let x = proposal(rng);
        let u: f64 = rng.gen_range(0.0..1.0);
        if u <= target(x) / (m * proposal_pdf(x)) {
            return x;
        }
    }
}
```

The efficiency is the ratio of areas: ∫f(x)dx / (M · ∫g(x)dx) = 1/M. For M close to 1 (tight envelope), few rejections. For M large, many rejections. The art is choosing g and M.

## Low-Discrepancy Sequences (Quasi-Monte Carlo)

For integration (not simulation), low-discrepancy sequences (Halton, Sobol, Faure) converge at O((log n)^s / n) vs. O(1/√n) for random Monte Carlo. For a 10-dimensional integral, quasi-MC with 10⁶ points achieves the same accuracy as random MC with 10⁹ points — a 1000× reduction.

```rust
fn sobol(dimension: usize, index: usize) -> f64 {
    let mut result = 0usize;
    let mut i = index;
    let mut j = 0usize;
    while i > 0 {
        if i & 1 != 0 { result ^= SOBOL_DIRECTION[j][dimension]; }
        i >>= 1;
        j += 1;
    }
    result as f64 / (1u64 << 32) as f64
}
```

The Sobol sequence uses precomputed direction numbers and XOR operations — it's faster than a PRNG and gives more accurate integrals. Use it whenever you control the sample points (numerical integration, not simulation of random processes).

## When to Use What

| Task | Method | Speed |
|------|--------|-------|
| Uniform [0,1) | Bitwise f64 construction | 5B/s |
| Normal(0,1) | Ziggurat | 4B/s |
| Exponential | Inverse CDF (-ln(u)/λ) | 1.5B/s |
| Gamma, Beta, Poisson | Rejection or specialized algorithms | 0.1–1B/s |
| Multivariate Normal | Cholesky + Ziggurat | O(d²) setup, then O(d) per sample |
| Numerical integration | Sobol quasi-MC | Faster convergence than MC |
| MCMC proposal | Uniform or Normal (Ziggurat) | Same as above |
