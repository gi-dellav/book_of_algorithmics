# Montgomery Multiplication

Peter Montgomery's 1985 paper "Modular Multiplication Without Trial Division" solved a critical bottleneck in cryptographic computation. The idea: transform numbers into a representation where the expensive modulo operation becomes cheap bitwise masking and shifting.

## The Problem

Modular multiplication `(a × b) mod n` requires:
1. A full-width multiply: `p = a × b` (2n bits for n-bit inputs).
2. A full-width division: `p mod n` (finding the remainder).

Division is 10–40× slower than multiplication. For RSA (2048-bit numbers), each multiplication produces 4096-bit intermediates, and the modulo is a 2048-bit division. This is devastating for performance.

## The Montgomery Representation

Choose R > n such that gcd(R, n) = 1, and R is a power of 2 (typically R = 2^k where k is the bit-width, e.g., 2^2048 for 2048-bit RSA).

**Montgomery form of a**: `a' = a × R mod n`

Addition in Montgomery form: `a' + b' = (a + b) × R mod n` — just add, then (if needed) subtract n.

Multiplication in Montgomery form: `a' × b' = (a × b) × R² mod n` — this is R× too large. We need a "Montgomery reduction" that divides by R modulo n.

## Montgomery Reduction (REDC)

Given T (0 ≤ T < nR), compute `T × R^(−1) mod n`:

```
function REDC(T):
    m = (T mod R) × n' mod R    // n' = -n^(-1) mod R
    t = (T + m × n) / R
    if t ≥ n: return t − n
    else: return t
```

The magic: `n'` is precomputed such that `n × n' ≡ −1 (mod R)`. Then `T + m × n` is divisible by R (because `T + m × n ≡ T + (T × n' mod R) × n ≡ T + T × (−1) ≡ 0 mod R`). The division by R (a power of two) is just a right shift — no hardware division needed.

The multiplication `m × n` is full-width, but `m < R` and R is typically the word size (2^64), so on a 64-bit machine this is a 64×64→128 multiply, which is fast.

**Computing n'**: n' = −n^(−1) mod R. Since R = 2^64, we need the modular inverse of n modulo 2^64. This can be computed with a few iterations of a doubling trick (Newton-like for modular inverse):

```rust
// Compute n' = -n^(-1) mod 2^64
fn montgomery_nprime(n: u64) -> u64 {
    let mut inv: u64 = 1;
    for _ in 0..5 {
        inv = inv.wrapping_mul(2u64.wrapping_sub(n.wrapping_mul(inv)));
    }
    inv.wrapping_neg()  // n' = (2^64 - inv) mod 2^64
}
```

After 5 iterations, `inv` is the exact inverse of `n` modulo 2^64 (and the loop can stop at 3 for 2^8, 4 for 2^16, etc.).

## Montgomery Multiplication

```rust
// Compute (a * b) * R^(-1) mod n  (REDC of the product)
fn montmul(a: u64, b: u64, n: u64, nprime: u64) -> u64 {
    let t_full: u128 = a as u128 * b as u128;
    let m: u64 = (t_full as u64).wrapping_mul(nprime);  // Low 64 bits of T * nprime
    let mut t: u128 = (t_full + (m as u128) * (n as u128)) >> 64;
    if t >= n as u128 { t -= n as u128; }
    t as u64
}
```

Three multiplications: `a × b` (128-bit), `T_low × nprime` (64-bit), `m × n` (128-bit). Plus two additions, a shift, and a conditional subtraction. All are fast — no division.

## Full Montgomery Modular Multiplication

To compute `(a × b) mod n`:

1. **Convert to Montgomery form**: `a' = montmul(a, R² mod n, n, nprime)` = `a × R × R² × R^(-1) = a × R mod n`. (Precompute `R² mod n` once.)
2. **Multiply in Montgomery form**: `c' = montmul(a', b', n, nprime)` = `(aR)(bR)R^(-1) = (ab)R mod n`.
3. **Convert back**: `c = montmul(c', 1, n, nprime)` = `(ab)R × 1 × R^(-1) = ab mod n`.

The two conversions each cost one Montgomery multiplication. If you do many multiplications (e.g., binary exponentiation with 2048-bit numbers), the conversion cost is amortized.

## Performance (Zen 2, 30-bit prime)

| Operation | Time (ns) |
|-----------|-----------|
| Direct modular multiplication (with hardware `div`) | ~15 |
| Montgomery multiplication | ~8 |
| Modular inverse (binary exponentiation, direct) | ~170 |
| Modular inverse (binary exponentiation, Montgomery) | ~158 |

The ~7% speedup for modular inverse hides the fact that each *multiplication* within the exponentiation is ~2× faster. For 2048-bit RSA, Montgomery is essentially always used.

## `constexpr` Montgomery

C++20 enables compile-time precomputation of `nprime` and `R² mod n`:

```rust
const fn compute_nprime_const(n: u64) -> u64 {
    let mut inv: u64 = 1;
    let mut i = 0;
    while i < 5 {
        inv = inv.wrapping_mul(2u64.wrapping_sub(n.wrapping_mul(inv)));
        i += 1;
    }
    inv.wrapping_neg()
}

struct Montgomery<const N: u64>;

impl<const N: u64> Montgomery<N> {
    const R2: u64 = ((1u128 << 128) % N as u128) as u64;  // R² mod N
    const NPRIME: u64 = compute_nprime_const(N);
    
    fn mul(a: u64, b: u64) -> u64 {
        let t_full: u128 = a as u128 * b as u128;
        let m: u64 = (t_full as u64).wrapping_mul(Self::NPRIME);
        let mut t: u128 = (t_full + (m as u128) * (N as u128)) >> 64;
        if t >= N as u128 { t -= N as u128; }
        t as u64
    }
}
```

For fixed-modulus arithmetic (common in finite field libraries), this eliminates the conversion overhead entirely — the compiler bakes `R²` and `nprime` into the code as constants.
