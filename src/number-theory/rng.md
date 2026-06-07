# Random Number Generation

Randomness is essential for cryptography, simulations, randomized algorithms, and Monte Carlo methods. Computers are deterministic — generating randomness requires either sampling a physical entropy source or running a pseudo-random generator that stretches a small seed into a long stream of seemingly-random bits.

## True Randomness: Hardware RNG

Modern CPUs have a hardware random number generator:

```c
uint64_t x;
_rdrand64_step(&x);  // Intel/AMD: true random number
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

```c
uint64_t xorshift128plus(uint64_t s[2]) {
    uint64_t s0 = s[0];
    uint64_t s1 = s[1];
    s[0] = s1;
    s0 ^= s0 << 23;
    s1 ^= s1 >> 18;
    s1 ^= s0 ^ (s0 >> 5);
    s[1] = s1;
    return s0 + s1;
}
```

V8 (Chrome's JavaScript engine) used xorshift128+ for `Math.random()`. Throughput: ~1 cycle per 64-bit output. Period: 2^128 − 1. Statistical quality: good enough for simulations, not for cryptography.

### PCG (Permuted Congruential Generator)

Melissa O'Neill's PCG combines an LCG with a permutation function that scrambles the output:

```c
uint32_t pcg32(uint64_t *state) {
    uint64_t oldstate = *state;
    *state = oldstate * 6364136223846793005ull + 1442695040888963407ull;
    uint32_t xorshifted = ((oldstate >> 18u) ^ oldstate) >> 27u;
    uint32_t rot = oldstate >> 59u;
    return (xorshifted >> rot) | (xorshifted << ((-rot) & 31));
}
```

PCG produces higher-quality output than xorshift at similar speed. Used in NumPy, Haskell, and many game engines.

### ChaCha20

A stream cipher that can also serve as a high-quality PRNG:

```c
void chacha20_block(uint32_t state[16], uint64_t counter, uint8_t output[64]) {
    // 20 rounds of quarter-round operations (add-rotate-XOR)
    // Produces 64 bytes of pseudo-random output per invocation
}
```

ChaCha20 is cryptographic quality, constant-time, and fast (~0.5 cycles/byte with AVX2). Used as the PRNG in Linux's `/dev/urandom` since kernel 4.8. Recommended when you need both speed and cryptographic quality.

## SIMD PRNGs

Generate multiple random numbers in parallel:

```c
// Generate 8 independent streams of xorshift
__m256i s0, s1;
for (int i = 0; i < n; i += 8) {
    s0 = _mm256_slli_epi64(s0, 23);
    // ... xorshift256+ ...
    __m256i result = _mm256_add_epi64(s0, s1);
    _mm256_storeu_si256((__m256i*)(out + i), result);
}
```

SIMD PRNGs can saturate memory bandwidth: 8 streams × 8 bytes × 2 GHz = 128 GB/s of pseudo-random output on a single core. For Monte Carlo simulations where the PRNG is the bottleneck, SIMD is essential.

## Seeding

The golden rule: **seed from a high-entropy source, then never again.**

```c
uint64_t seed;
_rdrand64_step(&seed);  // Hardware entropy

uint64_t state = seed;
// Optionally: mix with address space layout, current time, PID for defense-in-depth
state = wyhash(&seed, sizeof(seed), 0);
```

For reproducible research, log the seed. If your simulation finds a surprising result, you can re-run with the same seed to verify.

## Common Mistakes

1. **Seeding with `time(NULL)`**: Anyone who knows the approximate time you ran can reproduce your "random" sequence.
2. **Using `rand()` for anything**: It's slow (global mutex on many implementations), has a short period (2^31), and poor quality.
3. **Using `% n` to get a number in [0, n)**: Introduces modulo bias. Use rejection sampling or `arc4random_uniform`.
4. **Re-seeding a PRNG for every call**: Defeats the statistical properties and wastes entropy.
5. **Using non-cryptographic PRNGs for cryptographic purposes**: An attacker who can observe output can predict future values.
