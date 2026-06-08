# LU Factorization and Pivoting

LU factorization writes a square matrix A as the product of a lower triangular matrix L and an upper triangular matrix U: A = LU. Once factored, solving Ax = b reduces to two triangular solves (Ly = b, Ux = y) in O(n²) time. LU is the default factorization for non-symmetric, non-SPD matrices.

## Gaussian Elimination (Scalar)

```rust
fn lu_scalar(a: &mut [f64], n: usize) {
    for k in 0..n {
        let pivot = a[k * n + k];
        // Compute multipliers for column k below the diagonal
        for i in (k+1)..n {
            a[i * n + k] /= pivot;  // L[i][k]
        }
        // Update trailing submatrix
        for i in (k+1)..n {
            for j in (k+1)..n {
                a[i * n + j] -= a[i * n + k] * a[k * n + j];
            }
        }
    }
}
```

The inner two loops are a rank-1 update of the (n-k-1)×(n-k-1) trailing submatrix: O((n-k)²) FLOPs, O((n-k)²) memory — memory-bound. Performance: ~3% of peak.

## Partial Pivoting

If the pivot `a[k][k]` is zero (or very small), division by it is catastrophic. **Partial pivoting** swaps rows to bring the largest element in the current column to the pivot position:

```rust
fn lu_pivot(a: &mut [f64], ipiv: &mut [usize], n: usize) {
    for k in 0..n {
        // Find pivot: max |a[i][k]| for i ≥ k
        let mut max_val = a[k * n + k].abs();
        let mut max_idx = k;
        for i in (k+1)..n {
            let val = a[i * n + k].abs();
            if val > max_val {
                max_val = val;
                max_idx = i;
            }
        }
        ipiv[k] = max_idx;

        // Swap rows k and max_idx
        if max_idx != k {
            for j in 0..n {
                a.swap(k * n + j, max_idx * n + j);
            }
        }

        let pivot = a[k * n + k];
        for i in (k+1)..n { a[i * n + k] /= pivot; }
        for i in (k+1)..n {
            for j in (k+1)..n {
                a[i * n + j] -= a[i * n + k] * a[k * n + j];
            }
        }
    }
}
```

Partial pivoting guarantees numerical stability for most practical matrices (the growth factor is bounded by 2^{n-1} in the worst case, but practically < 10 for any non-pathological matrix). The cost: O(n²) comparisons and O(n²) row swaps — negligible compared to the O(n³) factorization.

## Block LU (BLAS-3)

The blocked LU algorithm processes blocks of size nb and uses BLAS-3 for the trailing submatrix update:

```rust
fn block_lu(a: &mut [f64], ipiv: &mut [usize], n: usize, nb: usize) {
    for k in (0..n).step_by(nb) {
        let kb = (k + nb).min(n) - k;

        // Factorize the panel A[k:n][k:k+kb] (BLAS-2, but small: n×kb)
        lu_panel(&mut a[k * n + k..], ipiv, kb, n - k, n);

        // Apply row interchanges to the left of the panel
        for i in k..k+kb {
            if ipiv[i] != i {
                for j in 0..k {
                    a.swap(i * n + j, ipiv[i] * n + j);
                }
            }
        }

        // Update the trailing submatrix: A[k+kb:n][k+kb:n]
        //   = A[...] - L[k+kb:n][k:k+kb] * U[k:k+kb][k+kb:n]
        //
        // dtrsm: solve L * U = A (compute U rows from panel)
        // dgemm: C = C - L_panel * U_panel (update trailing matrix)
        unsafe {
            dtrsm_(
                b"L", b"L", b"N", b"U",
                &(kb as i32), &((n - k - kb) as i32),
                &1.0, &a[k * n + k], &(n as i32),
                &mut a[k * n + k + kb], &(n as i32),
            );
            dgemm_(
                b"N", b"N",
                &((n - k - kb) as i32), &((n - k - kb) as i32), &(kb as i32),
                &(-1.0),
                &a[(k + kb) * n + k], &(n as i32),
                &a[k * n + k + kb], &(n as i32),
                &1.0,
                &mut a[(k + kb) * n + k + kb], &(n as i32),
            );
        }
    }
}
```

The `dgemm` call processes an (n-k-kb)×(n-k-kb) trailing submatrix with BLAS-3: O((n-k)² × kb) FLOPs on O((n-k)² + kb×(n-k)) memory. For large k, this is compute-bound. Near the end (k ≈ n), the trailing matrix is small and the panel factorization (BLAS-2) dominates.

Performance: ~24 GFLOPS for n = 4000 (75% of peak), compared to ~1 GFLOPS for the scalar version. The 24× speedup mirrors the matrix multiplication case study — and it's the same technique (blocking + BLAS-3).

## Solving with LU

After factorization, solve Ax = b in two triangular steps:

```rust
fn lu_solve(lu: &[f64], ipiv: &[usize], b: &mut [f64], n: usize) {
    // Step 1: Apply row interchanges to b
    for i in 0..n {
        if ipiv[i] != i {
            b.swap(i, ipiv[i]);
        }
    }

    // Step 2: Forward substitution Ly = Pb (L is unit lower triangular)
    // The multipliers are stored in the lower part of LU
    for i in 0..n {
        for j in 0..i {
            b[i] -= lu[i * n + j] * b[j];
        }
        // L[i][i] = 1, so no division needed
    }

    // Step 3: Back substitution Ux = y
    for i in (0..n).rev() {
        for j in (i+1)..n {
            b[i] -= lu[i * n + j] * b[j];
        }
        b[i] /= lu[i * n + i];
    }
}
```

Two O(n²) triangular solves. For n = 4000, the LU factorization takes ~10 ms and the solve takes ~0.2 ms. For 100 right-hand sides, factorize once (~10 ms) and solve 100 times (~20 ms) — the factorization is amortized.

## When LU Works (And When It Doesn't)

LU with partial pivoting is the most general factorization — it works on any non-singular square matrix. But:

- **Singular or near-singular matrices**: Pivot becomes zero or tiny. LAPACK reports an error. Use SVD for a rank-revealing decomposition.
- **Symmetric Positive Definite**: LU works but wastes 2× the FLOPs of Cholesky. Use `dpotrf` instead.
- **Symmetric Indefinite**: LDLᵀ with Bunch-Kaufman pivoting (2×2 pivots for stability) is preferred over LU.
- **Rectangular matrices**: LU requires square matrices. Use QR for tall/skinny, LQ for short/wide.

## The Performance-Stability Tradeoff

The blocked LU algorithm uses **partial pivoting within the panel** but not across the entire trailing submatrix. This is called **incremental pivoting** or **threshold pivoting**. It's numerically stable for almost all practical matrices, but there are rare pathological cases where it fails. LAPACK's `dgetrf` uses a hybrid: partial pivoting on the panel, then swapping rows in the trailing submatrix only if needed.

The alternative is **tournament pivoting** (used in communication-avoiding LU): select b pivot candidates per block, then run a tournament to choose the best. This reduces communication (O(n²) vs. O(n³) message count) at the cost of slightly worse pivot quality. For matrices where element growth is bounded, tournament pivoting is stable and 2–3× faster on distributed-memory systems.

## Benchmark Summary (n = 4000, double precision)

| Implementation | Time | GFLOPS | % of peak |
|---------------|------|--------|-----------|
| Scalar LU | 2.1 s | 1.0 | 3.1% |
| LAPACK dgetrf (1 core) | 45 ms | 47 | 147%* |

*LAPACK counts FLOPs differently — it uses fused multiply-add (2 FLOPs per FMA). Peak is 32 GFLOPS double-precision FMA. dgetrf achieves ~24 GFLOPS = 75% of peak.
