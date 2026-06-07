# Hash Functions

A hash function maps arbitrary data to a fixed-size output (the "digest" or "hash"). For performance work, we care about two categories: non-cryptographic hashes (fast, for hash tables and checksums) and cryptographic hashes (secure, for signatures and commitments).

## Non-Cryptographic Hash Functions

These are designed for speed, not security. Used in hash tables, Bloom filters, checksums, and data deduplication.

### Requirements

- **Uniform distribution**: Outputs should be evenly spread across the output range for typical inputs.
- **Avalanche effect**: Changing one input bit should flip ~50% of output bits.
- **Speed**: Should be faster than the operations that use the hash (memory access, network transfer).

No requirement for collision resistance, preimage resistance, or any cryptographic property. An attacker can find collisions easily; that's fine for hash tables.

### Common Algorithms

**FNV-1a** (Fowler-Noll-Vo): Simple multiplier-XOR chain.
```rust
fn fnv1a(data: &[u8], h: u64) -> u64 {
    let mut h = h;
    for &byte in data {
        h ^= byte as u64;
        h = h.wrapping_mul(0x100000001b3);  // FNV prime
    }
    h
}
```
~2 cycles per byte on Zen 2. Good for short keys. Poor at bulk hashing (not SIMD-friendly).

**xxHash64**: Designed for speed with SIMD support.
~0.5 cycles per byte, ~50 GB/s throughput on a single core. Uses 4 independent accumulators (ILP) + mix step. Good for bulk hashing.

**wyhash**: Modern design emphasizing speed and statistical quality.
~0.3 cycles per byte. Uses the `mum` (multiply-mix) primitive extensively. The hash table case studies (`data-structures/hash-tables.md`) use wyhash.

**SipHash**: Designed to be secure against hash-flooding DoS attacks while being fast enough for hash tables.
~1 cycle per byte. Used by Python, Rust, and many other languages for their default hash table implementations.

## Cryptographic Hash Functions

These are designed for security: collision resistance (hard to find two inputs with the same hash), preimage resistance (hard to find an input given a hash), and second preimage resistance (hard to find a second input with the same hash as a given input).

**SHA-256**: The workhorse. 32-byte digests, ~11 cycles per byte on Zen 2 (with SHA-NI hardware acceleration). Used in Bitcoin, TLS, code signing.

**SHA-3 / Keccak**: Sponge construction, different internal structure from SHA-2. ~12 cycles per byte with hardware acceleration.

**BLAKE3**: Modern design, ~0.5 cycles per byte (SIMD + tree hashing). Designed for both security and speed. Used in some newer systems.

## Merkle-Damgård vs. Sponge

Most cryptographic hashes use one of two construction patterns:

**Merkle-Damgård** (MD5, SHA-1, SHA-2): Process the message in fixed-size blocks. The state is a fixed-size chaining value. Simple but vulnerable to length extension attacks (solved with HMAC).

**Sponge** (SHA-3, BLAKE3): Absorb the message into a state, squeezing out the output. More flexible (variable output length) and naturally resistant to length extension.

## Digital Signatures

A digital signature scheme uses a hash function and asymmetric cryptography:

1. Hash the message: `h = H(m)`.
2. Sign the hash: `sig = Sign(sk, h)`.
3. Verify: `Verify(pk, h, sig)` → true/false.

Why hash first? Signing a 4 MB file directly with RSA would require a 4 MB signature (RSA operates on numbers the size of the modulus). Hashing reduces it to 32 bytes (SHA-256). The signature is then computed on the hash.

Common signature schemes: RSA-PSS (integer factorization), ECDSA (elliptic curve discrete log), Ed25519 (modern Edwards curve, deterministic, fast).

## JWT and Blockchain Use Cases

**JWT** (JSON Web Tokens): A header, payload, and signature, base64-encoded. The signature is typically HMAC-SHA256 (symmetric) or RS256 (RSA-SHA256, asymmetric). Verifying a JWT requires computing one hash.

**Blockchain proof-of-work** (Bitcoin): Find a nonce such that `SHA-256(SHA-256(block_header || nonce)) < target`. This is hashing as a lottery — the more hashes you can compute per second, the more likely you are to win the block reward. This drove the development of ASIC SHA-256 miners (terahashes per second).

**Proof-of-stake** (Ethereum 2.0): Hash functions are still used (for signatures, randomness beacons, and state commitments) but not for competitive mining. Latency and throughput matter less than in proof-of-work.

## Hash Function Throughput Benchmarks (Zen 2, single core)

| Algorithm | Throughput (GB/s) | Cycles/byte |
|-----------|-------------------|-------------|
| xxHash64 | ~50 | ~0.5 |
| wyhash | ~70 | ~0.3 |
| SipHash-2-4 | ~20 | ~1 |
| SHA-256 (SHA-NI) | ~2 | ~11 |
| BLAKE3 (AVX2) | ~50 | ~0.5 |

Non-cryptographic hashes are 20–100× faster than cryptographic ones. For hash tables and data structures, use non-cryptographic hashes. For security, use cryptographic hashes — but the hardware acceleration (SHA-NI, AES-NI) narrows the gap when the data is being encrypted anyway.
