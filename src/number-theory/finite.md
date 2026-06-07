# Finite Fields

A **field** is a set with addition, subtraction, multiplication, and division (by non-zero elements) that satisfy the usual arithmetic laws. Familiar examples: the rational numbers ℚ, real numbers ℝ, complex numbers ℂ. But these are infinite. For computation, we need finite fields.

## Prime Fields GF(p)

The integers modulo a prime p form a field. Addition, subtraction, and multiplication are done modulo p. Division is multiplication by the modular inverse (computed via extended Euclid or Fermat).

```rust
// Arithmetic in GF(7):
// 3 + 5 = 8 ≡ 1 (mod 7)
// 3 - 5 = -2 ≡ 5 (mod 7)
// 3 × 5 = 15 ≡ 1 (mod 7)
// 3 / 5 = 3 × 3 = 9 ≡ 2 (mod 7)  [5^(-1) = 3 because 5×3 = 15 ≡ 1]
```

Prime fields are used in RSA, DSA, and most elliptic curve cryptography (ECDSA, Curve25519).

## Binary Fields GF(2^k)

The integers modulo 2^k do NOT form a field (2 has no inverse — 2 × anything is even, never 1 mod 2^k). But we can construct a field with 2^k elements using polynomials over GF(2).

Elements of GF(2^k): polynomials of degree < k with coefficients in {0, 1}.
- Addition: XOR (polynomial addition mod 2).
- Multiplication: polynomial multiplication, then reduced modulo an **irreducible polynomial** of degree k.

Example: GF(2^8) with irreducible polynomial x^8 + x^4 + x^3 + x + 1 (AES uses this):

```
a = x^7 + x^5 + x^3 + x + 1       (0b10101011 = 0xAB)
b = x^4 + x^2 + 1                 (0b00010101 = 0x15)

a + b = x^7 + x^5 + x^4 + x^3 + x^2 + x  (XOR: 0xAB ^ 0x15 = 0xBE)

a × b = (x^7 + x^5 + x^3 + x + 1)(x^4 + x^2 + 1)
      = x^11 + x^9 + x^8 + x^7 + x^6 + x^5 + x^4 + x^3 + x^2 + x + 1
      reduced mod (x^8 + x^4 + x^3 + x + 1) → result
```

## Log/Exp Tables for GF(2^k)

For small fields (k ≤ 16), precompute logarithm and exponentiation tables:

```rust
// GF(2^8) multiplication via log tables
static mut GF256_LOG: [u8; 256] = [0u8; 256];  // log_g(x) for x ≠ 0
static mut GF256_EXP: [u8; 512] = [0u8; 512];  // g^i for i in [0, 510]

fn gf256_init() {
    // Find a primitive element (generator) g
    let g: u8 = 3;  // In GF(2^8) with AES polynomial, 3 is primitive
    let mut val: u8 = 1;
    for i in 0..255 {
        unsafe { GF256_EXP[i] = val; }
        unsafe { GF256_LOG[val as usize] = i as u8; }
        val = gf256_mul_basic(val, g);  // Multiply by generator
    }
    for i in 255..510 {
        unsafe { GF256_EXP[i] = GF256_EXP[i - 255]; }  // Wrap for convenience
    }
}

fn gf256_mul(a: u8, b: u8) -> u8 {
    if a == 0 || b == 0 { return 0; }
    unsafe { GF256_EXP[GF256_LOG[a as usize] as usize + GF256_LOG[b as usize] as usize] }
}

fn gf256_inv(a: u8) -> u8 {
    unsafe { GF256_EXP[255 - GF256_LOG[a as usize] as usize] }  // a^(-1) = g^(255 - log(a))
}
```

Multiplication in 3 table lookups + 1 add — faster than the polynomial multiplication and reduction. Used in AES S-box implementation, Reed-Solomon error correction, and QR codes.

## Finite Fields via CPU Instructions

Modern x86 CPUs have instructions for carry-less multiplication:

```rust
// GF(2^k) multiplication using CLMUL (carry-less multiply)
use std::arch::x86_64::*;

unsafe fn gf_mul(a: __m128i, b: __m128i) -> __m128i {
    _mm_clmulepi64_si128(a, b, 0x00)  // Multiply low 64 bits
}
```

`pclmulqdq` (Carry-Less Multiplication Quadword) computes the product of two 64-bit polynomials over GF(2) — exactly the multiplication needed for binary fields. Available since Westmere (2010). Reduction after multiplication still requires software (or Barrett-style reduction in GF(2)[x]).

AES-NI provides hardware AES encryption, including the S-box (which uses GF(2^8) inversion). For general binary field arithmetic, you still need software, but `pclmulqdq` makes it ~10× faster than pure software.

## Applications of Finite Fields

- **Cryptography**: All of elliptic curve cryptography. The curve is defined over a finite field GF(p) or GF(2^k).
- **Error correction**: Reed-Solomon codes use GF(2^8) for encoding.
- **AES**: The S-box is GF(2^8) inversion followed by an affine transformation.
- **Secret sharing**: Shamir's scheme uses polynomial evaluation over GF(p).
- **Hash functions**: GHASH (used in AES-GCM) uses GF(2^128) multiplication.
- **Random number generation**: Mersenne Twister uses GF(2) vector operations.

## Roots of Unity and FFT in Finite Fields

The Number Theoretic Transform (NTT) is the finite-field analogue of the FFT. Instead of complex roots of unity e^(2πi/N), it uses primitive N-th roots of unity in GF(p), where p is chosen so that p−1 is divisible by N.

For p = 998244353 (a common choice), p−1 = 2^23 × 119 — N can be any power of two up to 2^23. This enables FFT-like algorithms for polynomial multiplication in finite fields, used in post-quantum cryptography and polynomial commitments.

The NTT uses the same butterfly pattern as FFT but replaces complex arithmetic with modular arithmetic. Performance is limited by modular multiplication; Montgomery multiplication is essential for the inner loop.
