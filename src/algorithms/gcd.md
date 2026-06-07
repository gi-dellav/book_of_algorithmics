# Binary GCD

The greatest common divisor is one of the oldest algorithms in computer science — Euclid's algorithm dates to ~300 BCE. Yet a recent optimization by Daniel Lemire and Ralph Corderoy produces a binary GCD that is ~2× faster than `std::gcd` in libstdc++. This article traces the evolution.

## Euclid's Algorithm

```c
uint64_t gcd_euclid(uint64_t a, uint64_t b) {
    while (b != 0) {
        uint64_t t = a % b;
        a = b;
        b = t;
    }
    return a;
}
```

Each iteration uses the `div` instruction — 17–44 cycles on Zen 2. After ~40 iterations (worst-case for 64-bit Fibonacci numbers), that's ~1600 cycles. The hot loop: `div` + `mov` + `test` + `jne` ≈ 20 cycles/iteration.

## Binary GCD (Stein, 1967)

The binary GCD replaces division with subtraction and bit shifts:

```c
uint64_t gcd_binary(uint64_t a, uint64_t b) {
    if (a == 0) return b;
    if (b == 0) return a;
    
    int shift = __builtin_ctzll(a | b);  // Common factor of 2
    a >>= __builtin_ctzll(a);            // Remove factors of 2 from a
    
    do {
        b >>= __builtin_ctzll(b);        // Remove factors of 2 from b
        if (a > b) { uint64_t t = a; a = b; b = t; }
        b -= a;
    } while (b != 0);
    
    return a << shift;
}
```

Key properties:
- `gcd(a, b) = gcd(a, b − a)` (subtraction preserves gcd).
- `gcd(a, b) = 2 × gcd(a/2, b/2)` if both a and b are even.
- `gcd(a, b) = gcd(a/2, b)` if a is even and b is odd (and vice versa).

The loop eliminates all factors of 2 from both numbers, then subtracts the smaller from the larger. The subtraction guarantees at least one more factor of 2 can be extracted. Each iteration reduces the number of bits by at least 1 (often more), so it converges in O(log N) iterations.

No division instruction. The loop body: `tzcnt` + `shr` + `cmp` + `cmov` + `sub` + `test` + `jne` ≈ 5 cycles/iteration. Already ~4× faster than Euclid.

## The Lemire-Corderoy Optimization

The bottleneck in binary GCD is the loop control and the `if (a > b)` swap. Lemire and Corderoy observed that the swap can be eliminated by tracking which variable is "current":

```c
uint64_t gcd_lemire(uint64_t a, uint64_t b) {
    if (a == 0) return b;
    if (b == 0) return a;
    
    int shift = __builtin_ctzll(a | b);
    a >>= __builtin_ctzll(a);
    b >>= __builtin_ctzll(b);
    
    while (a != b) {
        if (a < b) {
            uint64_t t = a; a = b; b = t;
        }
        a -= b;
        a >>= __builtin_ctzll(a);
    }
    
    return a << shift;
}
```

Further optimizations:
- Unroll the loop 2× to increase ILP.
- Use `cmov` for the swap instead of a branch.
- Align the loop to a 16-byte boundary for optimal fetch.

**Performance on Zen 2:**
- `std::gcd` (libstdc++, Euclid): ~15 ns (30 cycles) for random 64-bit inputs.
- Binary GCD (Stein): ~6 ns (12 cycles).
- Lemire-Corderoy (optimized): ~4 ns (8 cycles).

~3.8× faster than `std::gcd`. The function is 20 lines of C. The speedup comes entirely from instruction selection (no `div`) and loop micro-optimization.

## The Deeper Insight

`std::gcd` uses Euclid because it's simpler and works well enough for most use cases. The libstdc++ maintainers could implement Lemire-Corderoy, but the performance of gcd is rarely anyone's bottleneck. This case study illustrates a general principle: standard library functions are designed for correctness and generality, not peak performance. When a function becomes your bottleneck, replacing it with a specialized implementation can yield significant speedups.

## When Binary GCD Wins

Binary GCD is always faster than Euclid on x86-64 because `tzcnt` + `sub` is always faster than `div`. On ARM (which doesn't have a hardware `div` in ARMv7 and earlier), binary GCD is dramatically faster. On RISC-V (which may not have hardware `div` either), binary GCD is the only viable option.

The binary GCD also extends naturally to:
- **GCD of arbitrary-precision integers**: The subtraction and shifting operations are O(n) for n-word integers; division is O(n²). Binary GCD is asymptotically better.
- **GCD in GF(2^k)** (polynomial GCD): The same "divide out the factor and subtract" logic works for polynomials, where the "factor of 2" becomes "factor of x."
