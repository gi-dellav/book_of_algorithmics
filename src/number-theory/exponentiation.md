# Binary Exponentiation

Binary exponentiation (also called exponentiation by squaring) computes `a^b mod m` in O(log b) multiplications. It's the fundamental building block of RSA, Diffie-Hellman, and primality testing.

## The Algorithm

Naive: multiply a by itself b times → O(b) operations. For b = 2^2048 (RSA), that's the age of the universe.

Binary: square repeatedly, multiplying by a when the corresponding bit of b is 1.

```
a^13 = a^(1101₂) = a^8 × a^4 × a^1
```

Start with `result = 1, base = a`. For each bit of b (from LSB to MSB):
- If bit is 1: `result = result × base mod m`
- `base = base² mod m`

This requires log₂(b) squarings and (on average) log₂(b)/2 multiplications. For 2048-bit RSA, that's ~3072 modular multiplications — fast enough to run in microseconds.

## Recursive Implementation

```rust
fn modpow_rec(a: u64, b: u64, m: u64) -> u64 {
    if b == 0 { return 1 % m; }
    let half = modpow_rec(a, b / 2, m);
    let mut result = ((half as u128) * (half as u128) % (m as u128)) as u64;
    if b & 1 != 0 {
        result = ((result as u128) * (a as u128) % (m as u128)) as u64;
    }
    result
}
```

## Iterative Implementation

```rust
fn modpow(a: u64, b: u64, m: u64) -> u64 {
    let mut result = 1 % m;
    let mut base = a % m;
    let mut b = b;
    while b != 0 {
        if b & 1 != 0 {
            result = ((result as u128) * (base as u128) % (m as u128)) as u64;
        }
        base = ((base as u128) * (base as u128) % (m as u128)) as u64;
        b >>= 1;
    }
    result
}
```

The `(__int128)` cast ensures the multiplication doesn't overflow before the modulo — essential when m > 2^32.

## Performance on Zen 2

Computing a modular inverse `a^(p−2) mod p` for a 30-bit prime:

- Naive (b multiplications): ~330 ns (benchmarked with b ≈ 10^9)
- Binary exponentiation: ~170 ns (30 squarings + ~15 multiplications)
- Extended Euclid (next article): ~135 ns

Binary exponentiation is the baseline. Extended Euclid is ~20% faster for modular inverse specifically. But binary exponentiation generalizes to any exponent, not just inverses.

## Montgomery Modular Multiplication

The inner loop of `modpow` does many modular multiplications. Each one does a 128-bit multiply followed by a 64-bit division (the modulo). The division is 10–40× slower than the multiply.

Montgomery multiplication (covered in detail in `montgomery.md`) eliminates the modulo operation by representing numbers in "Montgomery space" where reduction is done with bitwise operations and multiplication, not division. The speedup is dramatic: ~158 ns per modular inverse with Montgomery + binary exponentiation, vs. ~170 ns without.

## Constant-Time Considerations

For cryptographic use, `modpow` must run in **constant time** (no secret-dependent branching). Otherwise, an attacker can measure the timing of exponentiation and extract the secret exponent b.

Constant-time binary exponentiation:
```rust
fn modpow_ct(a: u64, b: u64, m: u64) -> u64 {
    let mut result = 1 % m;
    let mut base = a % m;
    let mut b = b;
    while b != 0 {
        // Always multiply, but conditionally use the result
        let tmp = ((result as u128) * (base as u128) % (m as u128)) as u64;
        result = if (b & 1) != 0 { tmp } else { result };  // cmov, not branch
        base = ((base as u128) * (base as u128) % (m as u128)) as u64;
        b >>= 1;
    }
    result
}
```

The ternary compiles to `cmov` (not a branch), assuming the compiler doesn't "optimize" it. Check the assembly. For serious cryptography, use a vetted library (OpenSSL, libsodium, BoringSSL) — constant-time programming has subtle pitfalls.
