# Random Number Generation

Randomness is essential for cryptography, simulations, randomized algorithms, and Monte Carlo methods. Computers are deterministic — generating randomness requires either sampling a physical entropy source or running a pseudo-random generator that stretches a small seed into a long stream of seemingly-random bits.

## True Randomness: Hardware RNG

Modern CPUs have a hardware random number generator:

```rust
let mut x: u64 = 0;
unsafe { std::arch::x86_64::_rdrand64_step(&mut x); }  // Intel/AMD: true random number
```

`rdrand` samples thermal noise from the CPU die, conditioning it with AES-CBC-MAC to remove bias. Throughput: ~100 MB/s (limited by entropy rate). Quality: true randomness, suitable for seeding cryptographic PRNGs.

There is also `rdseed` (slower, higher-quality entropy, directly from the entropy source rather than conditioned). Use `rdrand` for most purposes; `rdseed` for seeding where you need maximum entropy.

## Pseudo-Random Number Generators (PRNGs)

A PRNG produces a deterministic sequence that appears random. Given the same seed, it always produces the same sequence. PRNGs are faster than hardware RNGs and are used for simulations, randomized algorithms, and games.

### Linear Congruential Generator (LCG)

The simplest and oldest PRNG:

```
X_{n+1} = (a × X_n + c) mod m
```

Where a, c, m are carefully chosen constants. glibc's `rand()` uses this (with m = 2^31, a = 1103515245, c = 12345).

Problems:
- **Short period**: At most m (2^31 for 32-bit LCG).
- **Bad distribution in high dimensions**: Points (X_n, X_{n+1}, X_{n+2}) lie on a small number of hyperplanes.
- **Predictable**: After seeing a few outputs, an attacker can recover the full state.

Use LCGs only for non-cryptographic purposes where speed is paramount and quality barely matters (e.g., filling test data). Never for cryptography.

### Xorshift / Xorshift128+

George Marsaglia's xorshift family uses only XOR and shift operations:

```rust
fn xorshift128plus(s: &mut [u64; 2]) -> u64 {
    let mut s0 = s[0];
    let mut s1 = s[1];
    s[0] = s1;
    s0 ^= s0 << 23;
    s1 ^= s1 >> 18;
    s1 ^= s0 ^ (s0 >> 5);
    s[1] = s1;
    s0.wrapping_add(s1)
}
```

V8 (Chrome's JavaScript engine) used xorshift128+ for `Math.random()`. Throughput: ~1 cycle per 64-bit output. Period: 2^128 − 1. Statistical quality: good enough for simulations, not for cryptography.

### PCG (Permuted Congruential Generator)

Melissa O'Neill's PCG combines an LCG with a permutation function that scrambles the output:

```rust
fn pcg32(state: &mut u64) -> u32 {
    let oldstate = *state;
    *state = oldstate.wrapping_mul(6364136223846793005).wrapping_add(1442695040888963407);
    let xorshifted = (((oldstate >> 18) ^ oldstate) >> 27) as u32;
    let rot = (oldstate >> 59) as u32;
    (xorshifted >> rot) | (xorshifted << ((rot.wrapping_neg()) & 31))
}
```

PCG produces higher-quality output than xorshift at similar speed. Used in NumPy, Haskell, and many game engines.

### ChaCha20

A stream cipher that can also serve as a high-quality PRNG:

```rust
fn chacha20_block(state: &mut [u32; 16], counter: u64, output: &mut [u8; 64]) {
    // 20 rounds of quarter-round operations (add-rotate-XOR)
    // Produces 64 bytes of pseudo-random output per invocation
}
```

ChaCha20 is cryptographic quality, constant-time, and fast (~0.5 cycles/byte with AVX2). Used as the PRNG in Linux's `/dev/urandom` since kernel 4.8. Recommended when you need both speed and cryptographic quality.

## SIMD PRNGs

Generate multiple random numbers in parallel:

```rust
use std::arch::x86_64::*;

// Generate 8 independent streams of xorshift
let mut s0: __m256i;
let mut s1: __m256i;
unsafe {
    s0 = _mm256_setzero_si256();
    s1 = _mm256_setzero_si256();
    for i in (0..n).step_by(8) {
        s0 = _mm256_slli_epi64::<23>(s0);
        // ... xorshift256+ ...
        let result = _mm256_add_epi64(s0, s1);
        _mm256_storeu_si256(out.add(i) as *mut __m256i, result);
    }
}
```

SIMD PRNGs can saturate memory bandwidth: 8 streams × 8 bytes × 2 GHz = 128 GB/s of pseudo-random output on a single core. For Monte Carlo simulations where the PRNG is the bottleneck, SIMD is essential.

## Seeding

The golden rule: **seed from a high-entropy source, then never again.**

```rust
let mut seed: u64 = 0;
unsafe { std::arch::x86_64::_rdrand64_step(&mut seed); }  // Hardware entropy

let mut state = seed;
// Optionally: mix with address space layout, current time, PID for defense-in-depth
state = wyhash(&seed, std::mem::size_of::<u64>(), 0);
```

For reproducible research, log the seed. If your simulation finds a surprising result, you can re-run with the same seed to verify.

## Common Mistakes

1. **Seeding with `time(NULL)`**: Anyone who knows the approximate time you ran can reproduce your "random" sequence.
2. **Using `rand()` for anything**: It's slow (global mutex on many implementations), has a short period (2^31), and poor quality.
3. **Using `% n` to get a number in [0, n)**: Introduces modulo bias. Use rejection sampling or `arc4random_uniform`.
4. **Re-seeding a PRNG for every call**: Defeats the statistical properties and wastes entropy.
5. **Using non-cryptographic PRNGs for cryptographic purposes**: An attacker who can observe output can predict future values.
