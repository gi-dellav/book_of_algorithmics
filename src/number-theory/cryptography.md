# Cryptography

Cryptography is the largest-scale consumer of number-theoretic computation on the planet. Every TLS handshake, every secure email, every software update verification runs cryptographic algorithms that reduce to modular arithmetic, finite field operations, and hash function evaluation. This article surveys the key algorithms and their performance characteristics.

## Symmetric vs. Asymmetric

**Symmetric cryptography**: Same key for encryption and decryption. Fast (GB/s). Used for bulk data encryption.
- AES (Advanced Encryption Standard): 128/192/256-bit keys, 128-bit blocks.
- ChaCha20: 256-bit key, stream cipher.
- Hardware acceleration: AES-NI gives ~1 cycle/byte; ChaCha20 with AVX2 gives ~0.5 cycles/byte.

**Asymmetric cryptography**: Public key for encryption, private key for decryption (or signing/verification). Slow (operations per second, not GB/s). Used for key exchange and signatures.
- RSA: Based on integer factorization. 2048-bit keys are standard.
- ECDH/ECDSA: Based on elliptic curve discrete log. 256-bit keys provide equivalent security to 3072-bit RSA.
- Ed25519: Modern Edwards curve, fast, constant-time.

## RSA

### Key Generation

1. Generate two large primes p, q (typically 1024 bits each for 2048-bit RSA).
2. Compute n = pq (the modulus).
3. Compute φ(n) = (p−1)(q−1).
4. Choose e (typically 65537 = 2^16+1, a Fermat prime — small, fast to exponentiate).
5. Compute d = e^(−1) mod φ(n) (private exponent, using extended Euclid).

### Encryption/Decryption

- **Encrypt**: `c = m^e mod n` (public key: n, e).
- **Decrypt**: `m = c^d mod n` (private key: n, d).

Both are modular exponentiations. Encryption is fast (e is small — 17 squarings + 1 multiplication). Decryption is slow (d is large — ~3072 multiplications for 2048-bit RSA).

**Chinese Remainder Theorem (CRT) optimization**: Decrypt using p and q separately, then combine with CRT. This is ~4× faster than computing `c^d mod n` directly.

### Security

RSA security rests on the hardness of factoring n = pq. The best known algorithm (Number Field Sieve) has sub-exponential complexity. 2048-bit RSA is estimated to provide ~112 bits of security. Quantum computers running Shor's algorithm break RSA in polynomial time — this is driving the migration to post-quantum cryptography.

## Elliptic Curve Cryptography

ECC provides equivalent security to RSA with much smaller keys. An elliptic curve over GF(p) is the set of points (x, y) satisfying `y² = x³ + ax + b` (mod p) plus a point at infinity. The points form a group under "point addition."

**ECDH** (Elliptic Curve Diffie-Hellman):
- Alice generates private key a, public key A = aG (G is the generator point).
- Bob generates private key b, public key B = bG.
- Shared secret: aB = abG = bA = baG.

The core operation is **scalar multiplication**: compute kG for a point G and scalar k. This uses double-and-add (analogous to binary exponentiation) with ~256 doublings and ~128 additions for a 256-bit scalar.

**Curve25519** (Bernstein, 2006): A carefully designed curve with `y² = x³ + 486662x² + x` over GF(2^255 − 19). Designed for:
- Fast constant-time implementation (no branching on secret data).
- No need for point validation (Montgomery ladder inherently avoids small-subgroup attacks).
- Compact keys (32 bytes).

Ed25519 (Edwards form of the same curve) is used for signatures. Performance: ~50,000 signatures/second/core, ~20,000 verifications/second/core on Zen 2.

## TLS 1.3

The handshake that secures HTTPS:

1. Client → Server: Supported cipher suites, random nonce, DH key share.
2. Server → Client: Chosen cipher suite, random nonce, DH key share, certificate (containing public key), signature over the handshake.
3. Both sides derive session keys from the DH shared secret.
4. All subsequent data is encrypted with AES-GCM or ChaCha20-Poly1305.

Handshake cost: ~1 ECDH scalar multiplication + 1 signature verification (server side) + 1 signature generation (client side, if client cert used). On Zen 2, ~0.1 ms for a fresh handshake, ~0.01 ms for session resumption (using pre-shared keys, no asymmetric crypto).

## AES and Hardware Acceleration

AES operates on a 4×4 byte state matrix. Each round: SubBytes (S-box lookup, GF(2^8) inversion), ShiftRows (permutation), MixColumns (GF(2^8) matrix multiplication), AddRoundKey (XOR). AES-128 does 10 rounds.

Without hardware:
- S-box lookup: 256-byte table, 16 lookups/round → cache pressure.
- MixColumns: complex GF(2^8) arithmetic.
- Throughput: ~200 MB/s on a single core.

With AES-NI:
```c
__m128i aes_encrypt(__m128i plaintext, __m128i key) {
    __m128i state = plaintext;
    state = _mm_aesenc_si128(state, round_keys[0]);  // 1 round per instruction
    // ... 9 more rounds ...
    state = _mm_aesenclast_si128(state, round_keys[10]);
    return state;
}
```
Throughput: ~4 GB/s on a single core — 20× faster than software.

The hardware does each round in a single instruction (~4 cycles latency, throughput 1/cycle on Zen 2). This is a perfect example of the book's thesis: understanding the ISA (AES-NI exists) and using it (intrinsics) gives you a 20× speedup that no algorithmic improvement could match.

## Post-Quantum Cryptography

NIST is standardizing post-quantum algorithms to replace RSA and ECC when quantum computers become viable. Finalists include:

- **Kyber** (lattice-based key encapsulation): Based on Learning With Errors (LWE) over polynomial rings. Uses NTT (Number Theoretic Transform) heavily — this is where our finite field and Montgomery multiplication techniques apply directly.
- **Dilithium** (lattice-based signatures): Similar math, different construction.
- **SPHINCS+** (hash-based signatures): No number theory at all — uses only hash functions. Simpler but larger signatures.

The transition will be slow (decades), but the algorithms we've covered in this chapter are the building blocks of the post-quantum future.
