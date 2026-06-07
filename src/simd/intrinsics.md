# SIMD Intrinsics

When auto-vectorization fails — or when you need control over the exact instructions — use intrinsics. Intrinsics are C functions that map directly to SIMD instructions, giving you assembly-level control without writing assembly.

## Vector Types

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::{
    // 128-bit SSE
    __m128,    // 4 × f32
    __m128d,   // 2 × f64
    __m128i,   // 16 × i8, 8 × i16, 4 × i32, 2 × i64

    // 256-bit AVX/AVX2
    __m256,    // 8 × f32
    __m256d,   // 4 × f64
    __m256i,   // 32 × i8, 16 × i16, 8 × i32, 4 × i64

    // 512-bit AVX-512
    __m512,    // 16 × f32
    __m512d,   // 8 × f64
    __m512i,   // 64 × i8, 32 × i16, 16 × i32, 8 × i64
};
```

## Naming Convention

```
_mm<width>_<operation>_<suffix>

width:  (empty) = 128-bit, 256 = 256-bit, 512 = 512-bit
suffix: ps = packed single, pd = packed double
        epi32 = extended packed integer 32-bit
        si256 = scalar integer 256-bit
```

Examples:
- `_mm_add_ps`: Add packed singles (4 floats, 128-bit).
- `_mm256_add_epi32`: Add packed 32-bit integers (8 ints, 256-bit).
- `_mm512_fmadd_ps`: Fused multiply-add packed singles (16 floats, 512-bit).

## Basic Operations

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::{__m256, _mm256_loadu_ps, _mm256_add_ps, _mm256_storeu_ps};

// Array addition with AVX2
unsafe fn add_arrays_avx2(a: &[f32], b: &[f32], c: &mut [f32], n: usize) {
    for i in (0..n).step_by(8) {
        let va = _mm256_loadu_ps(&a[i]);  // Unaligned load
        let vb = _mm256_loadu_ps(&b[i]);
        let vc = _mm256_add_ps(va, vb);
        _mm256_storeu_ps(&mut c[i], vc);   // Unaligned store
    }
}
```

## GCC Vector Extensions

A more readable alternative to intrinsics:

```rust
// Rust nightly equivalent of GCC vector extensions:
#![feature(portable_simd)]
use std::simd::Simd;

type V8sf = Simd<f32, 8>;  // 32 bytes = 8 f32s

let va = V8sf::from_slice(&a[i..]);
let vb = V8sf::from_slice(&b[i..]);
let vc = va + vb;  // Looks like normal code, generates vaddps
vc.copy_to_slice(&mut c[i..]);
```

GCC vector extensions support operators (+, -, *, /, &, |, ^, ~) directly, making the code cleaner than intrinsics. However, they don't support all operations (shuffles, permutations, comparison predicates need intrinsics). Use as a bridge between auto-vectorization and full intrinsics.

## Portability Libraries

Writing intrinsics ties your code to a specific ISA. High-level SIMD libraries abstract the ISA:

- **Highway** (Google): C++ library targeting x86, ARM NEON, RISC-V V, WASM SIMD.
- **EVE** (Boost candidate): Expression templates for SIMD in C++.
- **VCL** (Agner Fog): Comprehensive vector class library, includes complex operations.
- **xsimd**: Lightweight SIMD wrapper for C++.

```rust
// Rust equivalent using portable_simd (nightly):
#![feature(portable_simd)]
use std::simd::Simd;

fn add(a: &[f32], b: &[f32], c: &mut [f32]) {
    const LANES: usize = 8;
    for i in (0..a.len()).step_by(LANES) {
        let va = Simd::<f32, 8>::from_slice(&a[i..]);
        let vb = Simd::<f32, 8>::from_slice(&b[i..]);
        (va + vb).copy_to_slice(&mut c[i..]);
    }
}
```

The same code compiles to SSE, AVX2, AVX-512, or NEON depending on the target. If you're writing a library, use Highway or similar — raw intrinsics are for when you need the absolute last 5% of performance.

## When to Use Each Level

| Level | Effort | Portability | Performance | Use Case |
|-------|--------|-------------|-------------|----------|
| Auto-vectorization | Zero | Excellent | Good | Most loops |
| GCC vector extensions | Low | Good (GCC/Clang) | Very good | Clean SIMD code |
| Intrinsics | Medium | x86 only | Excellent | Hot loops, crypto, codecs |
| Assembly | High | One CPU | Maximum | BLAS kernels, video codecs |

Start at the top and descend only when profiling shows a gap.
