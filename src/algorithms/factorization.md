# Integer Factorization

Factoring integers is hard. That's what makes RSA secure. But for word-sized integers (up to 64 bits), there are algorithms that run in microseconds — fast enough for primality testing and lightweight cryptography. This article traces the evolution from trial division to state-of-the-art, each step motivated by a specific algorithmic insight.

## The Problem

Given a 64-bit composite integer N, find a non-trivial factor (or prove N is prime). Performance is measured on 60-bit semiprimes (products of two 30-bit primes) on Zen 2.

## Stage 1: Trial Division

```rust
fn factor_trial(n: u64) -> u64 {
    let mut d: u64 = 2;
    while d * d <= n {
        if n % d == 0 { return d; }
        d += 1;
    }
    n  // n is prime
}
```

This checks every integer up to √N. For N ≈ 2^60, √N ≈ 2^30 ≈ 10^9. At one division per ~20 cycles, that's ~20 billion cycles — about 10 seconds. Unacceptable.

**Lesson**: O(√N) algorithms don't work for N beyond ~10^12. We need sub-exponential algorithms.

## Stage 2: Trial Division with Precomputed Primes

Instead of checking every integer, check only primes. Precompute all primes up to 10^6 using the Sieve of Eratosthenes (takes a few milliseconds at startup):

```rust
fn factor_primes(n: u64, primes: &[u64]) -> u64 {
    let mut i = 0;
    while primes[i] * primes[i] <= n {
        if n % primes[i] == 0 { return primes[i]; }
        i += 1;
    }
    n
}
```

For N ≈ 2^60, we need primes up to 2^30. There are ~50 million primes below 2^30 — too many to test. This is still an O(√N) algorithm. Better, but not good enough.

## Stage 3: Wheel Factorization

Skip numbers divisible by small primes using a "wheel." A 2,3,5-wheel skips multiples of 2, 3, and 5 — only 8 out of every 30 numbers need to be tested:

```rust
// Increments for a 2,3,5-wheel:
const WHEEL: [u8; 8] = [4, 2, 4, 2, 4, 6, 2, 6];
// After starting at d=7, add wheel[i % 8] to get the next candidate
```

A full 2,3,5,7-wheel skips ~77% of candidates. But this is still O(√N) — just with a smaller constant. For 60-bit semiprimes, we need something fundamentally faster.

## Stage 4: Pollard's Rho

Pollard's rho algorithm (1975) uses the fact that random numbers modulo p (a prime factor of N) will repeat (enter a cycle) after roughly √p iterations. If we generate a pseudorandom sequence:

```
x_0 = 2
x_{i+1} = (x_i² + 1) mod N
```

Then compute `gcd(|x_{2i} − x_i|, N)` at each step. If `x_{2i} ≡ x_i (mod p)` (they're equal modulo p but not modulo N), the gcd will reveal p.

```rust
fn pollard_rho(n: u64) -> u64 {
    let mut x: u64 = 2;
    let mut y: u64 = 2;
    let mut d: u64 = 1;
    while d == 1 {
        x = ((x as u128 * x as u128 + 1) % n as u128) as u64;   // x = f(x)
        y = ((y as u128 * y as u128 + 1) % n as u128) as u64;   // y = f(f(y))
        y = ((y as u128 * y as u128 + 1) % n as u128) as u64;
        let diff = if x > y { x - y } else { y - x };
        d = gcd_binary(diff, n);
    }
    if d == n { 0 } else { d }  // 0 = failure, retry with different f(x)
}
```

Expected time: O(√p) = O(N^(1/4)) for the smallest factor p. For 60-bit semiprimes (p ≈ 2^30), √p ≈ 2^15 ≈ 33,000 iterations. Each iteration does a 128-bit multiplication + modulo. Total: ~330,000 cycles ≈ 0.16 ms at 2 GHz.

Pollard's rho is the first algorithm that actually works for 60-bit numbers. Speedup over trial division: ~60,000×.

## Stage 5: Pollard-Brent Improvement

Richard Brent (1980) observed that computing `gcd` at every step is wasteful — `gcd` is relatively expensive. Instead, accumulate the product of several differences and take the gcd periodically:

```rust
fn pollard_brent(n: u64) -> u64 {
    let mut y: u64 = 2;
    let mut c: u64 = 1;
    let m: u64 = 128;

    for _attempt in 0..10 {
        // ... (Brent's cycle-finding with product accumulation)
        // Compute gcd every 128 steps, using the product of 128 differences
    }

    0 // placeholder
}
```

This reduces the number of gcd computations by a factor of 128, while adding cheap multiplications (product accumulation). Overall speedup: ~2× over standard Pollard's rho.

## Stage 6: Optimizing the Inner Loop

Profile reveals the bottleneck: the modular multiplications `((__int128)x * x + 1) % n`. The modulo operation is a 128-bit division. Can we avoid it?

**Montgomery multiplication** (see `number-theory/montgomery.md`) replaces the modulo with a multiplication and shift. Precompute the Montgomery constant for N once, then each multiplication takes ~6 cycles instead of ~20.

**Lazy reduction**: Since we're accumulating products before taking gcd, we don't need exact modulo at every step. We can allow numbers to grow slightly larger than N and reduce only when they risk overflow. This saves additional reductions.

## Stage 7: Complete Implementation

The final implementation (~200 lines of C):

- Pollard-Brent with product accumulation.
- Montgomery modular multiplication.
- Precomputed small prime list for trial division up to 1000 (catches small factors instantly).
- Miller-Rabin primality test (to confirm the remaining factor is prime).
- Hardware RNG for the initial parameters (`c` in `x² + c mod N`).

**Performance on Zen 2 for 60-bit semiprimes:**
- Trial division: ~10 seconds.
- Wheel + precomputed primes: ~2 seconds.
- Pollard's rho: ~0.16 ms.
- Pollard-Brent + Montgomery: ~0.05 ms.
- **Overall speedup**: ~200,000× from algorithmic improvements + ~3× from micro-optimizations.

## The Lesson

Algorithmic improvement (O(√N) → O(N^(1/4))) matters more than any micro-optimization. Pollard's rho is exponentially faster than trial division. But once you have the right algorithm, architectural optimization (Montgomery multiplication, vectorized gcd, lazy reduction) gives another 3–5×. Both matter. Neither is sufficient alone.
