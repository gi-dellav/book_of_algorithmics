# Extended Euclidean Algorithm

Euclid's algorithm computes the greatest common divisor of two integers. The **extended** Euclidean algorithm also finds the coefficients x and y such that `ax + by = gcd(a, b)`. When gcd(a, m) = 1, the coefficient x is the modular inverse of a modulo m. This is faster than Fermat exponentiation and works for any coprime modulus, not just primes.

## Euclid's Algorithm

The basic insight: `gcd(a, b) = gcd(b, a mod b)`. Iterate until b = 0:

```rust
fn gcd(mut a: u64, mut b: u64) -> u64 {
    while b != 0 {
        let t = a % b;
        a = b;
        b = t;
    }
    a
}
```

Each iteration reduces the larger number by at least half (the modulus operation `a mod b` produces a result less than b), so the number of iterations is O(log min(a,b)). The algorithm is remarkably fast — ~5–10 iterations for 64-bit numbers.

## The Extended Version

Maintain invariants as we compute gcd:

```
a = original_a × a_coeff_a + original_b × a_coeff_b
b = original_a × b_coeff_a + original_b × b_coeff_b
```

When b reaches 0, a = gcd, and the coefficients give us the linear combination.

```rust
// Returns gcd(a,b) and (x,y) such that ax + by = gcd(a,b)
fn egcd(a: i64, b: i64) -> (i64, i64, i64) {
    if b == 0 {
        return (a, 1, 0);
    }
    let (g, x1, y1) = egcd(b, a % b);
    (g, y1, x1 - (a / b) * y1)
}
```

## Iterative Version (Faster, No Recursion)

```rust
fn modinv(mut a: i64, m: i64) -> i64 {
    let mut t = 0;
    let mut newt = 1;
    let mut r = m;
    let mut newr = a;
    
    while newr != 0 {
        let q = r / newr;
        let tmp = t;
        t = newt;
        newt = tmp - q * newt;
        let tmp = r;
        r = newr;
        newr = tmp - q * newr;
    }
    
    // r is now gcd(a, m). If r > 1, no inverse exists.
    if t < 0 { t += m; }
    t  // t is a^(-1) mod m
}
```

This tracks only the coefficients we care about (the coefficient of `a`). It's ~10% faster than the recursive version due to loop unrolling and register allocation by the compiler.

## Performance Comparison (Zen 2, 30-bit modulus)

| Method | Time (ns) | Notes |
|--------|-----------|-------|
| Fermat (modpow) | ~170 | Simple, works for prime modulus only |
| Extended Euclid (iterative) | ~135 | ~20% faster, works for any coprime modulus |
| Montgomery + Euclid | ~158 | Full Montgomery transform + Euclid inverse |

Extended Euclid is the fastest general-purpose method for modular inverse. Montgomery provides the fastest *repeated* operations with the same modulus (by transforming to Montgomery space once, then doing many fast operations). We cover Montgomery in the next article.

## When Modular Inverse Is Needed

- **RSA**: Computing the private exponent `d = e^(−1) mod φ(n)`.
- **Elliptic curve crypto**: Point addition formula requires division (which is modular inverse).
- **Chinese Remainder Theorem**: Combining residues uses modular inverses of the moduli.
- **Polynomial interpolation**: Lagrange interpolation needs modular inverses.
- **Fractional arithmetic in finite fields**: `a/b = a × b^(−1)`.

## Binary GCD

An alternative to Euclid's algorithm that uses only subtraction and shifting (no division):

```rust
fn binary_gcd(mut a: u64, mut b: u64) -> u64 {
    if a == 0 { return b; }
    if b == 0 { return a; }
    
    let shift = (a | b).trailing_zeros();  // Common powers of 2
    a >>= a.trailing_zeros();
    
    loop {
        b >>= b.trailing_zeros();
        if a > b { std::mem::swap(&mut a, &mut b); }
        b -= a;
        if b == 0 { break; }
    }
    
    a << shift
}
```

The binary GCD is ~2× faster than Euclid's algorithm on x86-64 because:
- No division instruction (only shifts, subtraction, comparison).
- `tzcnt` (count trailing zeros) is a single-cycle instruction.
- The loop body is shorter and less branchy.

Chapter 11 (`algorithms/gcd.md`) develops a state-of-the-art binary GCD from Dan Lemire that's ~2× faster than `std::gcd`.
