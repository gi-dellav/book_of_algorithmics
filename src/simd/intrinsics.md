# SIMD Intrinsics

When auto-vectorization fails — or when you need control over the exact instructions — use intrinsics. Intrinsics are C functions that map directly to SIMD instructions, giving you assembly-level control without writing assembly.

## Vector Types

```c
#include <x86intrin.h>

// 128-bit SSE
__m128   v4float;    // 4 × float
__m128d  v2double;   // 2 × double
__m128i  vints;      // 16 × int8, 8 × int16, 4 × int32, 2 × int64

// 256-bit AVX/AVX2
__m256   v8float;    // 8 × float
__m256d  v4double;   // 4 × double
__m256i  vints;      // 32 × int8, 16 × int16, 8 × int32, 4 × int64

// 512-bit AVX-512
__m512   v16float;   // 16 × float
__m512d  v8double;   // 8 × double
__m512i  vints;      // 64 × int8, 32 × int16, 16 × int32, 8 × int64
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

```c
// Array addition with AVX2
void add_arrays_avx2(float *a, float *b, float *c, int n) {
    for (int i = 0; i < n; i += 8) {
        __m256 va = _mm256_loadu_ps(a + i);  // Unaligned load
        __m256 vb = _mm256_loadu_ps(b + i);
        __m256 vc = _mm256_add_ps(va, vb);
        _mm256_storeu_ps(c + i, vc);          // Unaligned store
    }
}
```

## GCC Vector Extensions

A more readable alternative to intrinsics:

```c
typedef float v8sf __attribute__((vector_size(32)));  // 32 bytes = 8 floats

v8sf va = *(v8sf *)(a + i);
v8sf vb = *(v8sf *)(b + i);
v8sf vc = va + vb;  // Looks like normal code, generates vaddps
*(v8sf *)(c + i) = vc;
```

GCC vector extensions support operators (+, -, *, /, &, |, ^, ~) directly, making the code cleaner than intrinsics. However, they don't support all operations (shuffles, permutations, comparison predicates need intrinsics). Use as a bridge between auto-vectorization and full intrinsics.

## Portability Libraries

Writing intrinsics ties your code to a specific ISA. High-level SIMD libraries abstract the ISA:

- **Highway** (Google): C++ library targeting x86, ARM NEON, RISC-V V, WASM SIMD.
- **EVE** (Boost candidate): Expression templates for SIMD in C++.
- **VCL** (Agner Fog): Comprehensive vector class library, includes complex operations.
- **xsimd**: Lightweight SIMD wrapper for C++.

```cpp
// Highway example
#include <hwy/highway.h>

namespace HWY_NAMESPACE {
void Add(const float *a, const float *b, float *c, size_t n) {
    const ScalableTag<float> d;
    for (size_t i = 0; i < n; i += Lanes(d)) {
        auto va = Load(d, a + i);
        auto vb = Load(d, b + i);
        Store(va + vb, d, c + i);
    }
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
