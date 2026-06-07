# Integer Division

Integer division is the slowest common arithmetic operation on modern CPUs. On Zen 2, a 64-bit integer division takes 17–44 cycles — 10–40× slower than multiplication. Compilers go to extraordinary lengths to avoid it. This article explains those lengths, and what to do when the compiler can't help.

## Why Division Is Slow

Multiplication is a grid of AND gates feeding a tree of adders. With enough silicon, it can be deeply pipelined (latency 3, throughput 1/cycle on Zen 2 for 64-bit multiply).

Division is iterative. The hardware implements a radix-4 or radix-8 SRT division algorithm that produces 2–3 bits of the quotient per cycle. A 64-bit division takes ~20 cycles at minimum, and the execution unit is not fully pipelined — you can't start a second division until the first finishes.

Floating-point division is also slow (13 cycles for single precision) but faster than integer because the mantissa is only 24 or 53 bits.

The lesson: **avoid division in hot code.** If you must divide, divide by a constant — the compiler can replace it with multiplication.

## Division by Constants: The Magic Number Trick

The compiler transforms `x / c` (where `c` is a compile-time constant) into `(x * magic) >> shift`. This is 3–5× faster than a hardware division.

For unsigned division:
```
x / c ≈ (x * m) >> (n + s)
```
where `m = ⌈2^(n+s) / c⌉` is a precomputed "magic number" and `s` is chosen to give enough precision.

```rust
// The compiler turns this:
fn div_by_10(x: u32) -> u32 { x / 10 }

// Into something like:
fn div_by_10(x: u32) -> u32 {
    // magic = 0xCCCCCCCD (for 32-bit unsigned)
    let t: u64 = ((x as u64) * 0xCCCCCCCDu64) >> 32;
    (t >> 3) as u32
}
```

The magic number `0xCCCCCCCD` is `⌈2^35 / 10⌉`. The product fits in 64 bits; the result is in the high 32 bits, shifted by 3 to account for the extra bits in the magic multiplier.

For signed division, the compiler adds a correction term to handle negative numbers correctly.

## Barrett Reduction

When the divisor is not a compile-time constant but you need to divide many numbers by the same divisor, you can precompute a magic number once and reuse it:

```rust
// Precompute magic for divisor d
fn precompute_barrett(d: u64) -> u64 {
    ((u128::MAX) / d as u128 + 1) as u64  // ceil(2^128 / d) without overflow
}

// Divide x by d using the precomputed magic
fn barrett_reduce(x: u64, d: u64, magic: u64) -> u64 {
    let t: u128 = (x as u128) * (magic as u128);
    let mut q: u64 = (t >> 64) as u64;  // Quotient = high 64 bits of product
    // q might be off by 1; correct if needed
    let mut r = x - q * d;
    if r >= d {
        q += 1;
        r -= d;
    }
    q  // or return r for modulo
}
```

The Barrett reduction computes `x / d` as `⌊x × (2^128 / d) / 2^128⌋`. The magic number `ceil(2^128 / d)` is computed once (using one hardware division). Each subsequent "division" is one 128-bit multiply, one shift, and possibly one correction. The multiply is 3 cycles; the whole operation is ~5 cycles — 4–8× faster than hardware division.

## Lemire Reduction (2019)

Daniel Lemire discovered an even faster reduction when both the division and modulo are needed, or when the numbers are small enough:

```rust
// Lemire's fast modulo: compute x % d without division
// Assumes d < 2^32 and we only need the remainder
fn lemire_mod(x: u32, d: u32) -> u32 {
    let magic: u64 = ((1u128 << 64) / d as u128 + 1) as u64;
    let lowbits: u64 = (x as u64) * magic;
    ((lowbits as u128 * d as u128) >> 64) as u32
}
```

This replaces the correction step (expensive branch) with a second multiplication. For 32-bit operands, it's both faster and branchless.

The key insight: the low 64 bits of `x * magic` contain enough information to compute the remainder, but we need to multiply by `d` again and take the high 64 bits. Two multiplications, no branches, ~6 cycles total.

## When Division Is Unavoidable

Sometimes you genuinely need division with a variable divisor (e.g., hash table probing, `a % b` where both are dynamic):

1. **Use 32-bit operands if possible.** 32-bit division is faster (~12 cycles) than 64-bit (~20+ cycles).
2. **Batch divisions**: If you have many divisions by the same divisor, precompute the Barrett/Lemire magic.
3. **Use floating-point division as an approximation**: `x / d ≈ to_int(to_float(x) / to_float(d))`. Accurate for integers up to ~2^24 for float, ~2^53 for double. Faster but requires careful correctness analysis.
4. **Use SIMD**: AVX-512 has vectorized integer division operations (AVX-512IFMA, AVX-512VAES). If you have AVX-512 hardware, div becomes 5× faster for vectors.
5. **Restructure the algorithm**: Can you replace `hash(k) % size` with `hash(k) & (size-1)` by using power-of-two table sizes? (Be aware of cache associativity issues — see Chapter 9.)

## Benchmark: Division vs. Multiply-Shift

On Zen 2, for 64-bit unsigned integers:

| Operation | Latency | Throughput |
|-----------|---------|------------|
| `div r64` | 17–44 | 17–44 |
| `imul r64, r64` | 3 | 1 |
| `shr r64, cl` | 1 | 0.5 |

The compiler's multiply-shift replacement:
```asm
mov rax, magic
mul rdi          ; rdx:rax = x * magic
shr rdx, shift   ; rdx = quotient
```
Total latency: ~5 cycles (mov + mul + shr). ~10× faster than `div`.

This is entirely the compiler's doing — just write `x / constant` and check the assembly to confirm the `div` is gone.
