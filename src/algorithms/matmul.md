# Matrix Multiplication

Matrix multiplication is the most studied algorithm in high-performance computing. It's also a perfect microcosm of this book: the naive implementation achieves 0.2 GFLOPS, and with cache blocking, SIMD, and register reuse — all in under 40 lines of C — we achieve 28 GFLOPS, a 140× speedup. Each stage targets a specific bottleneck.

## The Baseline

```rust
for i in 0..n {
    for j in 0..n {
        for k in 0..n {
            c[i][j] += a[i][k] * b[k][j];
        }
    }
}
```

n = 1024, single precision. Performance: ~0.2 GFLOPS. The `B[k][j]` access has stride n (4 KB) — every access is a cache miss. The loop is memory-bound on RAM latency, not compute.

## Stage 1: Loop Reordering (Cache Locality)

Swap j and k loops:

```rust
for i in 0..n {
    for k in 0..n {
        for j in 0..n {
            c[i][j] += a[i][k] * b[k][j];
        }
    }
}
```

Now the inner loop accesses `C[i][j]` and `B[k][j]` sequentially (contiguous memory). `A[i][k]` is constant within the inner loop — it can be hoisted. Performance: ~1.2 GFLOPS (6× speedup). The inner loop is now streaming through memory — bandwidth-bound rather than latency-bound.

## Stage 2: Vectorization

The inner loop is a dot product of `B[k][j]` and a scalar `A[i][k]`, stored into `C[i][j]`. The compiler can vectorize this (with `-march=native`):

```rust
// Compiler-generated AVX2 inner loop:
// vbroadcastss A[i][k] → ymm0
// vmovups B[k*stride + j] → ymm1
// vfmadd231ps C[i*stride + j], ymm0, ymm1 → C[i*stride + j]
```

Performance: ~6 GFLOPS (30× over baseline). Memory bandwidth is now the bottleneck: we read B (n² elements, sequentially), read A (n² elements, hoisted), and read/write C (n² elements, RMW). That's ~3n² words at 4 bytes each = 12 GB for n=1024.

## Stage 3: Register Reuse (Tiling in i and k)

The inner loop reloads `C[i][j]` for each k iteration. If we process a small tile of C (say, 6 rows × 16 columns), we can keep the tile in registers and accumulate into it:

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

let mut i = 0;
while i < n {
    let mut k = 0;
    while k < n {
        let mut j = 0;
        while j < n {
            // C[i:i+6][j:j+16] += A[i:i+6][k] * B[k][j:j+16]
            // C tile (6×16 = 96 floats) stays in YMM registers
            for ii in 0..6 {
                let vb = _mm256_loadu_ps(&B[k][j]);
                let aik = _mm256_broadcast_ss(&A[i + ii][k]);
                // FMA: C[ii][j:j+8] += aik * vb (8 floats at once)
            }
            j += 16;
        }
        k += 1;
    }
    i += 6;
}
```

The key: 6 × 16 = 96 floats = 12 YMM registers for the C tile, 1 register for `vb`, 1 for `aik` broadcast. Zen 2 has 16 YMM registers — it fits. The C tile is loaded once, accumulated into across all k iterations, and stored once at the end.

Performance: ~18 GFLOPS (90× over baseline). Memory traffic drops to just reading A and B (C stays in registers). The loop is now compute-bound.

## Stage 4: Cache Blocking (Tiling in k)

The i-k-j order means B is streamed sequentially, but A is loaded repeatedly (for each k). If n is large, A doesn't fit in cache and must be reloaded from RAM for each k iteration. Fix: tile in all three dimensions:

```rust
const BLOCK: usize = 48;

let mut ii = 0;
while ii < n {
    let mut kk = 0;
    while kk < n {
        let mut jj = 0;
        while jj < n {
            // Multiply C[ii:ii+B][jj:jj+B] += A[ii:ii+B][kk:kk+B] × B[kk:kk+B][jj:jj+B]
            for i in ii..ii + BLOCK {
                for k in kk..kk + BLOCK {
                    for j in jj..jj + BLOCK {
                        c[i][j] += a[i][k] * b[k][j];
                    }
                }
            }
            jj += BLOCK;
        }
        kk += BLOCK;
    }
    ii += BLOCK;
}
```

Choose BLOCK so that three BLOCK×BLOCK tiles fit in L1 cache: 3 × BLOCK² × 4 bytes ≤ 32 KB → BLOCK ≈ 52. Use 48 for even divisibility.

Performance: ~24 GFLOPS (120× over baseline). The inner BLOCK×BLOCK multiplication stays within L1 cache; the outer loop streams through B and A blocks that fit in L2/L3.

## Stage 5: Packing and Micro-Kernel

The inner micro-kernel (the BLOCK×BLOCK multiplication) can be further optimized by:
- Pre-packing A and B tiles into contiguous memory (eliminates strided loads).
- Unrolling the micro-kernel to use all 16 YMM registers optimally.
- Choosing tile sizes that are multiples of the SIMD width and the register count.

A typical micro-kernel: 6 rows × 16 columns of C, accumulating over a column of A and a row of B (both loaded once into registers). The inner loop is pure FMA:

```asm
vfmadd231ps ymm0, ymm8, ymm12    ; C[0][0:8] += A[0][k] * B[k][0:8]
vfmadd231ps ymm1, ymm8, ymm13    ; C[0][8:16] += A[0][k] * B[k][8:16]
vfmadd231ps ymm2, ymm9, ymm12    ; C[1][0:8] += A[1][k] * B[k][0:8]
...
; 12 FMA instructions (6 rows × 2 columns of 8 floats)
; Each FMA: 8 multiplies + 8 adds = 16 FLOPS
; 12 × 16 = 192 FLOPS per issue
; At 2 FMA/cycle on Zen 2, this saturates the FP pipes
```

Performance: ~28 GFLOPS (140× over baseline). ~87% of Zen 2's theoretical 32 GFLOPS peak. The remaining gap is from loop overhead, imperfect memory alignment, and the fact that we're not using AVX-512.

## Stage 6: Multi-Threading

The outer loops over ii and jj are independent — just add `#pragma omp parallel for` and scale to 8 cores. Near-linear scaling for large n (n ≥ 2048): ~200 GFLOPS on 8 Zen 2 cores.

## The Complete Code

```rust
const BLOCK: usize = 48;
const MICRO_H: usize = 6;
const MICRO_W: usize = 16;

unsafe fn matmul(c: *mut f32, a: *const f32, b: *const f32, n: usize) {
    let mut ii = 0;
    while ii < n {
        let mut kk = 0;
        while kk < n {
            let mut jj = 0;
            while jj < n {
                // Pack A[ii:ii+B][kk:kk+B] and B[kk:kk+B][jj:jj+B] into aligned buffers
                let mut i = ii;
                while i < ii + BLOCK {
                    let mut j = jj;
                    while j < jj + BLOCK {
                        micro_kernel(c, a_packed, b_packed, i, j, kk, BLOCK);
                        j += MICRO_W;
                    }
                    i += MICRO_H;
                }
                jj += BLOCK;
            }
            kk += BLOCK;
        }
        ii += BLOCK;
    }
}
```

Under 40 lines of C for the core logic. The packing routines and micro-kernel add ~100 lines. The result is within 10% of OpenBLAS for square matrices — close enough that only a hardware-specific assembly kernel can close the gap.

## What Each Stage Teaches

| Stage | Bottleneck | Technique | Chapter |
|-------|-----------|-----------|---------|
| Loop reorder | Cache misses | Locality | 8 (External Memory) |
| Vectorization | Scalar instructions | Auto-vectorization | 10 (SIMD) |
| Register tiling | Memory bandwidth | Multiple accumulators | 3 (Pipelining) |
| Cache blocking | L2/L3 misses | Cache-oblivious design | 9 (CPU Caches) |
| Micro-kernel | Port pressure | Instruction scheduling | 3 (Pipelining) |
| Multi-threading | Single core | Parallel decomposition | 13 (Parallel) |

This is the arc of the entire book, in one case study.
