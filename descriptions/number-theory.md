# Chapter: Number Theory (`number-theory/`)

## Overview

Weight 7 in the book. Opens with a G.H. Hardy quote about number theory being "useless" — proved wrong by cryptography. The chapter covers modular arithmetic, binary exponentiation, extended Euclidean algorithm, Montgomery multiplication, finite fields/Galois fields, and has drafts for hashing, cryptography, error correction, and random number generation. It sits naturally between the arithmetic chapter (IEEE 754, integer ops) and the algorithms chapter (factorization, GCD).

## Files and Content

| File | Status | Size | Description |
|------|--------|------|-------------|
| `_index.md` | Complete | 3.1 KB | Hardy's Apology framing: number theory went from "useless" to the foundation of cryptography |
| `modular.md` | Complete | 7.9 KB | Modular arithmetic: residues, congruence, day-of-week problem, Fermat's little theorem (with multinomial proof), Fermat primality test, modular inverse via $a^{p-2}$ |
| `exponentiation.md` | Complete | 4.7 KB | Binary exponentiation: recursive → iterative → fully unrolled. ~170ns per modular inverse on Zen 2. |
| `euclid-extended.md` | Complete | 4.1 KB | Extended Euclidean algorithm for modular inverse when modulus is not prime. Recursive → iterative. ~135ns, ~10ns faster than binary exponentiation. |
| `montgomery.md` | Published | 12.3 KB | Montgomery multiplication: Montgomery space, reduction algorithm (derived from $r·r^{-1} + n·n' = 1$ identity), fast inverse via doubling trick, complete `constexpr` implementation. ~158ns per inverse when fully optimized. |
| `finite.md` | Draft | 5.9 KB | Finite fields: permutation groups, roots of unity, complex numbers, Euler's formula, Galois fields GF(2^k) with log/exp tables for multiplication |
| `error-correction.md` | Draft/Stub | 59 B | Empty placeholder for "Error Correction" |
| `hashing.md` | Draft | 1.8 KB | Brief notes: hash function properties, cryptographic vs. non-cryptographic, digital signatures, JWT, blockchain proof-of-work |
| `cryptography.md` | Draft | 5.2 KB | RSA (key generation, encryption/decryption), man-in-the-middle, symmetric vs. asymmetric, AES (confusion/diffusion), one-time pads, quantum computing threats |
| `rng.md` | Draft/Stub | 161 B | "Linear congruent generator, period of LCG" — two sentences |

## Image Assets

1 image: `clock.gif` (2.3 KB) — a clock animation illustrating modular arithmetic (cyclic, small remainders).

## Strengths

1. **`montgomery.md` is the star**: This article is exceptionally well-written. The derivation from $r·r^{-1} + n·n' = 1$ to the final reduction formula is step-by-step and clear. The "move the right-shift earlier" optimization is a genuine insight. The complete `constexpr` implementation with inline benchmarks makes it immediately usable.
2. **Strong mathematical rigor**: Fermat's theorem is proved (via multinomial coefficients), the Montgomery identity is derived, and convergence rates are analyzed. The chapter doesn't just present algorithms; it justifies them.
3. **Good progression**: Modular arithmetic → exponentiation → extended Euclid → Montgomery builds naturally. Each article references the previous ones.
4. **Concrete performance numbers**: Every algorithm is benchmarked on Zen 2 with specific nanosecond counts (330ns → 180ns → 170ns → 158ns for modular inverse), showing the cumulative impact of each optimization.
5. **Hardy framing is memorable**: The historical irony of number theory powering cryptography is an effective hook.

## Areas for Improvement

1. **`finite.md` is incomplete**: The article jumps from complex numbers to Galois fields without a clear bridge. The polynomial representation and `log`/`ilog` tables for GF(2⁸) are shown but not explained. The article lacks benchmarks and practical applications.
2. **`cryptography.md` is messy**: It reads like lecture notes rather than a polished article. The RSA explanation is correct but mixed with incomplete thoughts about GPU tensor cores and stock prices. The man-in-the-middle section is a single sentence.
3. **`hashing.md` is scattered**: Covers hash functions, digital signatures, JWT, and blockchain in 1.8 KB — each topic gets ~1 paragraph. No concrete implementations or benchmarks.
4. **`error-correction.md` is empty**: Error-correcting codes (Reed-Solomon, Hamming, LDPC) have fascinating hardware implementation aspects and would fit well in this chapter.
5. **`rng.md` is a stub**: Random number generation deserves a proper treatment — LCG, PCG, xorshift, ChaCha20, and the SIMD acceleration of PRNGs.
6. **Missing connections to `algorithms/factorization.md`**: The factorization article is in a different chapter, but it heavily uses number theory. A cross-reference would strengthen both chapters.

## Recommendations

1. **Complete `finite.md`**: Add a clear progression: integers mod p → why we want fields of size 2^k → polynomial representation → irreducible polynomials → GF(2⁸) construction → log/exp table implementation → application to AES S-box. Include benchmarks comparing GF multiplication vs. table lookup.
2. **Rewrite `cryptography.md`**: Structure as: asymmetric (RSA → ECC), symmetric (AES), hashing (SHA), and protocols (TLS 1.3 handshake cost). Include cycle counts for AES-NI instructions.
3. **Consolidate `hashing.md`**: Focus on non-cryptographic hashing (MurmurHash3, xxHash, wyhash) with SIMD implementations and collision benchmarks. Move crypto hashing to `cryptography.md`.
4. **Write `error-correction.md`**: Cover Hamming codes (popcount-based syndrome), Reed-Solomon (GF(2⁸) from `finite.md`), and the performance of software vs. hardware (Intel ISA-L) implementations.
5. **Write `rng.md`**: Cover LCG (period, modulus choices), PCG, xorshift128+, and the throughput in GB/s when vectorized with SIMD. Mention `rdrand`/`rdseed` hardware instructions and their limitations.
6. **Add cross-references**: `modular.md` → `algorithms/factorization.md` (Pollard's rho uses modular arithmetic), `montgomery.md` → `algorithms/matmul.md` (exercise: efficient modular matrix multiplication).
