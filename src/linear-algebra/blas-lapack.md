# BLAS and LAPACK

The Basic Linear Algebra Subprograms (BLAS) and Linear Algebra PACKage (LAPACK) are the most successful software libraries in history. Every numerical computing system — NumPy, MATLAB, Julia, R — delegates to them. The secret is a three-level hierarchy that matches the hardware's compute-to-memory ratio.

## The Three-Level Hierarchy

| Level | Operation | FLOPs | Memory | Compute/Memory |
|-------|-----------|-------|--------|---------------|
| BLAS-1 | Vector-vector: `y = αx + y` | O(n) | O(n) | 2:1 |
| BLAS-2 | Matrix-vector: `y = αAx + βy` | O(n²) | O(n²) | 2:1 |
| BLAS-3 | Matrix-matrix: `C = αAB + βC` | O(n³) | O(n²) | O(n):1 |

The compute-to-memory ratio is the key. BLAS-1 and BLAS-2 are memory-bound: you touch each element of the input a fixed number of times, and the CPU sits idle waiting for data. BLAS-3 is compute-bound: for n = 1000, you do O(n³) = 10⁹ FLOPs on O(n²) = 10⁶ elements — a 1000:1 ratio. The hardware can achieve 80–95% of peak GFLOPS.

This is why LAPACK **blocks** everything into BLAS-3 operations. A Cholesky factorization that operates on single columns is BLAS-2 (memory-bound). A blocked Cholesky that processes 256×256 tiles is BLAS-3 (compute-bound). Same math, 10× speed difference.

## The BLAS Interface

BLAS functions follow a naming convention: `{prec}{name}` where `prec` is `s` (single), `d` (double), `c` (complex single), `z` (complex double). Example: `dgemm` = double-precision general matrix multiply.

```rust
// BLAS dgemm signature:
// C = α*op(A)*op(B) + β*C
// op = N (no transpose), T (transpose), C (conjugate transpose)
// Dimensions: A is m×k, B is k×n, C is m×n
extern "C" {
    fn dgemm_(
        transa: *const u8, transb: *const u8,
        m: *const i32, n: *const i32, k: *const i32,
        alpha: *const f64,
        a: *const f64, lda: *const i32,
        b: *const f64, ldb: *const i32,
        beta: *const f64,
        c: *mut f64, ldc: *const i32,
    );
}
```

The leading dimension parameters (`lda`, `ldb`, `ldc`) allow operating on submatrices of larger arrays — essential for blocked algorithms. A call to multiply two 1000×1000 matrices:

```rust
unsafe {
    let (m, n, k) = (1000i32, 1000i32, 1000i32);
    let (alpha, beta) = (1.0f64, 0.0f64);
    dgemm_(
        b"N" as *const u8, b"N" as *const u8,
        &m, &n, &k,
        &alpha,
        a.as_ptr(), &m,
        b.as_ptr(), &k,
        &beta,
        c.as_mut_ptr(), &m,
    );
}
```

Performance: ~28 GFLOPS on Zen 2 single core (peak: 32 GFLOPS for double-precision FMA). That's 87.5% of peak — the blocked algorithms in OpenBLAS squeeze nearly everything out of the hardware.

## LAPACK Factorization Ecosystem

LAPACK provides factorizations that solve `Ax = b` for different matrix types:

| Matrix type | Factorization | Function | Complexity |
|------------|--------------|----------|------------|
| General, square | LU with partial pivoting | `dgetrf` | (2/3)n³ |
| Symmetric Positive Definite | Cholesky | `dpotrf` | (1/3)n³ |
| Symmetric Indefinite | LDLᵀ with pivoting | `dsytrf` | (1/3)n³ |
| General, rectangular | QR | `dgeqrf` | (4/3)mn² |
| General (least squares) | SVD | `dgesvd` | O(mn²) |

The Cholesky factorization costs half the FLOPs of LU (1/3 n³ vs. 2/3 n³) — symmetry exploitation matters even at the algorithmic level, before any optimizations.

## Calling BLAS from Rust

The `blas` and `cblas` crates provide safe wrappers. The `ndarray` crate with the `blas` feature delegates to the system BLAS:

```rust
// ndarray-linalg or direct blas-sys
use blas::dgemm;

fn matmul(a: &[f64], b: &[f64], c: &mut [f64], n: usize) {
    unsafe {
        dgemm(
            b'N', b'N', n as i32, n as i32, n as i32,
            1.0, a, n as i32, b, n as i32,
            0.0, c, n as i32,
        );
    }
}
```

When to use BLAS vs. write your own:
- **Use BLAS** for matrices larger than ~100×100 where the tiling overhead pays off.
- **Write your own** for very small matrices (n < 64) where BLAS function call overhead dominates, or for custom hardware (GPU, FPGA) where the BLAS interface doesn't map well.
- **Use BLAS** whenever numerical stability matters — OpenBLAS and MKL have decades of edge-case testing.

## Installing a Fast BLAS

The system BLAS matters enormously. On Ubuntu:

```
# Netlib reference BLAS (~1% of peak performance on large matrices)
sudo apt install libblas-dev

# OpenBLAS (~85% of peak, multi-threaded)
sudo apt install libopenblas-dev

# BLIS (AMD-optimized, ~90% of peak)
# Build from source: github.com/flame/blis
```

The reference BLAS is for verification, not performance. OpenBLAS or BLIS are 50–100× faster on large matrices. The matrix multiplication case study in Chapter 11 achieved ~28 GFLOPS — OpenBLAS achieves ~30 GFLOPS on the same hardware, because it additionally uses prefetching, register allocation tuned per microarchitecture, and adaptive tiling based on cache size detection at runtime.

## The Blocked Cholesky Pattern

To understand LAPACK's performance, let's trace a blocked Cholesky factorization. For an SPD matrix A, we want L such that A = LLᵀ.

Column-oriented (BLAS-2, memory-bound):

```rust
// For each column j, compute L[j:n][j] = A[j:n][j] / sqrt(A[j][j])
// Then update A[j+1:n][j+1:n] -= L[j+1:n][j] * L[j+1:n][j]ᵀ
// This is a rank-1 update (BLAS-2: dsyr) — memory-bound
```

Blocked (BLAS-3, compute-bound):

```rust
// Process blocks of size nb (typically 256):
for j in (0..n).step_by(nb) {
    // Factorize the diagonal block (BLAS-2, but small: nb×nb)
    dpotrf_(&L[j..][j..]);

    // Solve L[j+nb:][j:j+nb] (BLAS-3: dtrsm)
    dtrsm_(&L[j..][j..], &L[j+nb..][j..]);

    // Update the trailing submatrix (BLAS-3: dsyrk)
    // C = C - A * Aᵀ  where C is (n-j-nb)×(n-j-nb), A is (n-j-nb)×nb
    dsyrk_(&mut A[j+nb..][j+nb..], &A[j+nb..][j..]);
}
```

The `dsyrk` (symmetric rank-k update) is a BLAS-3 operation: O(nb × (n-j)²) FLOPs on O((n-j)² + nb × (n-j)) memory. The ratio is O(nb):1 — large enough to be compute-bound for nb ≥ 64.

## The Cost of Not Blocking

For n = 4000, double precision:

| Algorithm | GFLOPS | % of peak |
|-----------|--------|-----------|
| Naive Cholesky (column-oriented) | 0.8 | 2.5% |
| BLAS-2 Cholesky (dsyr) | 3.2 | 10% |
| Blocked Cholesky (BLAS-3) | 26 | 81% |
| OpenBLAS dpotrf (multi-threaded, 8 cores) | 180 | 70% (of 256 peak) |

The difference between BLAS-2 and BLAS-3 Cholesky is 8×. The difference between naive and OpenBLAS is 225×. This is the same pattern we saw with matrix multiplication (140× from naive to optimized) — and it applies to every factorization in LAPACK.

The lesson: never write a factorization yourself. Always call LAPACK. And if you must write one, block it ruthlessly.
