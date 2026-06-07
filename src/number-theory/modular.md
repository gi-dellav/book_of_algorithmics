# Modular Arithmetic

Modular arithmetic is arithmetic where numbers "wrap around" at a fixed modulus. It's the arithmetic of clocks, cyclic groups, and cryptography. Every fast cryptographic primitive builds on it.

## Definition

```
a ≡ b (mod m)  ⟺  m divides (a − b)
```

Read: "a is congruent to b modulo m." The numbers a and b have the same remainder when divided by m.

Examples (mod 12):
- 15 ≡ 3 (mod 12) — 3 PM is 15:00.
- −2 ≡ 10 (mod 12) — 10 AM is two hours before noon.
- 12 ≡ 0 (mod 12).

## Residue Classes

The integers modulo m form m residue classes: `{..., −2m, −m, 0, m, 2m, ...}`, `{..., −2m+1, −m+1, 1, m+1, 2m+1, ...}`, etc. We represent each class by its canonical representative, typically 0 through m−1 (or sometimes −⌊(m−1)/2⌋ through ⌊m/2⌋ for convenience).

Computers use unsigned modular arithmetic natively: `uint32_t` arithmetic is modulo 2³². The wrap-around is guaranteed by the C standard for unsigned types.

## Basic Operations

All operations are modulo m:

- **Addition**: `(a + b) mod m`. On a computer, compute `a + b`, then if the sum ≥ m, subtract m. Or use a conditional move.
- **Subtraction**: `(a − b) mod m`. If the result is negative, add m.
- **Multiplication**: `(a × b) mod m`. The product can be up to m², requiring double-width arithmetic (`__int128` for 64-bit modulus).

```c
// Modular addition, branchless (assumes 0 ≤ a,b < m)
uint64_t mod_add(uint64_t a, uint64_t b, uint64_t m) {
    uint64_t sum = a + b;
    uint64_t diff = sum - m;
    return (sum >= m || sum < a) ? diff : sum;
    // The 'sum < a' handles the case where a+b overflowed (wrapped around)
}
```

## Fermat's Little Theorem

If p is prime and a is not divisible by p:

```
a^(p−1) ≡ 1 (mod p)
```

Corollary: `a^(p−2) ≡ a^(−1) (mod p)` — the modular inverse of a modulo a prime is a^(p−2). This is the basis of RSA and gives us a way to compute modular inverses when the modulus is prime.

Proof sketch: The set `{a, 2a, 3a, ..., (p−1)a}` modulo p is a permutation of `{1, 2, 3, ..., p−1}`. Multiplying all elements: `a^(p−1) × (p−1)! ≡ (p−1)! (mod p)`. Cancel `(p−1)!` (which is coprime to p) to get `a^(p−1) ≡ 1`.

## The Day-of-Week Problem

What day of the week was January 1, 2000?

- Known: January 1, 1900 was a Monday.
- Days between: 100 years × 365 days + 24 leap days = 36,524 days.
- 36,524 mod 7 = 5 (since 36,519 = 7 × 5,217, remainder 5).
- Monday + 5 days = Saturday.

This is modular arithmetic: we computed 36,524 mod 7 using the fact that the days of the week are integers modulo 7.

## Modular Inverse

The modular inverse of a modulo m is the number x such that `a × x ≡ 1 (mod m)`. It exists iff gcd(a, m) = 1.

Three methods:
1. **Fermat's little theorem**: `x = a^(m−2) mod m` (only works for prime m).
2. **Extended Euclidean algorithm**: Works for any coprime a, m. Faster than exponentiation.
3. **Precomputed tables**: If m is fixed and a is from a small set, store `a^(−1) mod m` in a table.

We cover methods 1 and 2 in the next two articles. Method 3 is the fastest when applicable (e.g., Montgomery multiplication uses precomputed inverses for the modulus).
