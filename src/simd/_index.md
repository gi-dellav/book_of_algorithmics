# Data Parallelism on x86

SIMD — Single Instruction, Multiple Data — is the biggest performance lever available on modern CPUs. A single AVX-512 instruction can do 16 floating-point operations in the same time a scalar instruction does one. This chapter teaches you to harness SIMD, from auto-vectorization to hand-coded intrinsics.

## The Speedup: A Preview

Summing an array of 1 million floats on Zen 2 (2 GHz):

| Method | Time | GFLOPS | Speedup |
|--------|------|--------|---------|
| Scalar (no flags) | 2.5 ms | 0.4 | 1× |
| Auto-vectorized (`-O2 -march=native`) | 0.6 ms | 1.7 | 4× |
| Intrinsics (AVX2, 2 acc) | 0.3 ms | 3.3 | 8× |
| Intrinsics (AVX2, 8 acc, unrolled) | 0.15 ms | 6.7 | 17× |

The first 4× speedup costs zero effort — just a compiler flag. The next 4× requires understanding intrinsics, accumulators, and loop unrolling. This chapter takes you through all of it.

## SIMD History and Extensions

| Extension | Year | Width | Float ops/inst | Introduced |
|-----------|------|-------|----------------|------------|
| MMX | 1996 | 64-bit | 2×float | Only integers |
| SSE | 1999 | 128-bit | 4×float | Pentium III |
| SSE2 | 2001 | 128-bit | 2×double | Pentium 4 |
| SSE4.2 | 2008 | 128-bit | + string/text | Nehalem |
| AVX | 2011 | 256-bit | 8×float | Sandy Bridge |
| AVX2 | 2013 | 256-bit | + integer SIMD | Haswell |
| AVX-512 | 2017 | 512-bit | 16×float | Skylake-X |
| AVX-512 (client) | 2021 | 512-bit | partial | Alder Lake (E-cores lack it) |

## AVX-512 and Downclocking

AVX-512 operations consume significant power. On some Intel processors (Skylake-X), using 512-bit instructions causes the core to downclock by 200–500 MHz. The aggregate throughput may actually *decrease* if the workload doesn't benefit enough from 512-bit width.

Zen 4 handles AVX-512 differently: it executes 512-bit instructions on two 256-bit units, taking two cycles. The clock penalty is minimal. Check your processor's behavior before assuming AVX-512 is always faster.

## The SIMD Programming Model

SIMD is data parallelism: the same operation is applied to multiple data elements simultaneously. A SIMD register holds a small fixed-size vector; SIMD instructions operate on entire vectors.

```rust
// Scalar:
for i in 0..n {
    c[i] = a[i] + b[i];
}

// SIMD conceptual model:
for i in (0..n).step_by(8) {
    let va = load_f32x8(&a[i..]);
    let vb = load_f32x8(&b[i..]);
    let vc = va + vb;  // 8 additions in one instruction
    store_f32x8(&mut c[i..], vc);
}
```

This chapter covers:
- **Auto-vectorization**: Getting the compiler to generate SIMD for you.
- **Intrinsics**: Writing SIMD in C using `_mm*` builtins.
- **Data movement**: Loads, stores, broadcasts, gather/scatter.
- **Masking**: Branchless conditionals in SIMD.
- **Reductions**: Summing, min/max, horizontal operations.
- **In-register shuffles**: Permuting vector elements for table lookups, filter, compress.
