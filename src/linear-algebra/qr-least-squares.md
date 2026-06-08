# QR Factorization and Least Squares

When you have more equations than unknowns (an overdetermined system), there is no exact solution. You want the **least squares** solution: minimize ‖Ax - b‖₂. QR factorization solves this with better numerical stability than the normal equations, and it's the foundation for eigenvalue algorithms (QR iteration).

## The QR Decomposition

Factor A (m×n, m ≥ n) into Q (m×m orthogonal) and R (m×n upper triangular). Since the bottom (m-n) rows of R are zero, the "economy" QR is: A = Q₁ R₁ where Q₁ is m×n with orthonormal columns and R₁ is n×n upper triangular.

The least squares solution: R₁ x = Q₁ᵀ b. Back-substitution on R₁ gives x.

## Householder Reflections

A Householder reflection H = I - 2vvᵀ (where ‖v‖₂ = 1) is a symmetric orthogonal matrix that maps any vector to a multiple of e₁. To zero out column k below the diagonal:

```rust
fn householder(x: &[f64]) -> (f64, Vec<f64>) {
    let n = x.len();
    let norm = x.iter().map(|&xi| xi * xi).sum::<f64>().sqrt();
    if norm == 0.0 { return (0.0, vec![0.0; n]); }

    let alpha = if x[0] > 0.0 { -norm } else { norm };
    let mut v = x.to_vec();
    v[0] -= alpha;

    let v_norm = v.iter().map(|&vi| vi * vi).sum::<f64>().sqrt();
    for vi in &mut v { *vi /= v_norm; }

    (alpha, v)
}

fn apply_householder(a: &mut [f64], v: &[f64], m: usize, n: usize, k: usize) {
    // Apply H = I - 2vvᵀ to columns k..n of A
    for j in k..n {
        let mut dot = 0.0f64;
        for i in k..m { dot += v[i - k] * a[i * n + j]; }
        dot *= 2.0;
        for i in k..m { a[i * n + j] -= dot * v[i - k]; }
    }
}
```

The Householder QR factorization applies n reflections to reduce A to upper triangular form:

```rust
fn qr_householder(a: &mut [f64], m: usize, n: usize) -> Vec<f64> {
    let mut tau = vec![0.0f64; n]; // store scalar factors for Q reconstruction

    for k in 0..n {
        // Extract column k below diagonal
        let mut x = vec![0.0f64; m - k];
        for i in k..m { x[i - k] = a[i * n + k]; }

        let (alpha, v) = householder(&x);
        tau[k] = 2.0 / (v[0] * v[0] + v[..].iter().map(|&vi| vi * vi).sum::<f64>());

        // Store alpha in R[k][k]
        a[k * n + k] = alpha;

        // Store v in the lower part of column k (for later Q reconstruction)
        for i in 1..v.len() { a[(k + i) * n + k] = v[i]; }

        // Apply reflection to trailing columns
        let v_full: Vec<f64> = {
            let mut vf = vec![1.0f64]; // v[0] = 1 (implicit in compact storage)
            vf.extend_from_slice(&v[1..]);
            vf
        };
        apply_householder_compact(a, &v_full, tau[k], m, n, k);
    }
    tau
}
```

This is O(mn²) operations. For m = n = 2000: ~40 ms on Zen 2 (scalar). The BLAS-2 nature (matrix-vector products in `apply_householder`) limits performance to ~15% of peak.

## Blocked QR

Just as LU and Cholesky, QR can be blocked. The key is the **compact WY representation**: accumulate b Householder reflections into a BLAS-3-friendly form. After processing a block of b columns, the accumulated transformation is:

```
Q_block = I - Y * T * Yᵀ
```

Where Y is (m-b)×b (the reflection vectors) and T is b×b upper triangular. Applying this to the trailing matrix is a `dgemm`:

```rust
// After accumulating b reflections:
// A[k+b: :][k+b:] = (I - Y * T * Yᵀ) * A[k+b:][k+b:]
// = A[k+b:][k+b:] - Y * (T * (Yᵀ * A[k+b:][k+b:]))
//                dgemm         dgemm              dgemm
```

Three `dgemm` calls — all BLAS-3, all compute-bound. Blocked QR achieves ~65% of peak GFLOPS for large matrices.

## Least Squares: QR vs. Normal Equations

The naive approach to least squares is the **normal equations**:

```
Aᵀ A x = Aᵀ b
```

Form C = Aᵀ A (symmetric positive semi-definite, n×n), then Cholesky factorize C, then solve. This is O(mn²) to form C + O(n³/3) to factorize. For m ≫ n, forming C is the bottleneck.

The problem: forming Aᵀ A squares the condition number. If κ(A) = 10⁸, κ(Aᵀ A) = 10¹⁶ — you lose all significant digits in double precision. The normal equations are numerically dangerous.

QR factorization solves this: ‖Ax - b‖₂ = ‖Rx - Qᵀ b‖₂ (since Q is orthogonal, it preserves the 2-norm). Compute y = Qᵀ b (O(mn²)), then solve Rx = y (O(n²) back-substitution). No condition number squaring.

```rust
fn least_squares_qr(a: &[f64], b: &[f64], m: usize, n: usize) -> Vec<f64> {
    let mut a_copy = a.to_vec();
    let tau = qr_householder(&mut a_copy, m, n);

    // Compute Qᵀ b
    let mut b_copy = b.to_vec();
    for k in 0..n {
        // Apply k-th Householder reflection to b
        let v: Vec<f64> = {
            let mut vf = vec![1.0f64];
            for i in 1..(m - k) { vf.push(a_copy[(k + i) * n + k]); }
            vf
        };
        let mut dot = b_copy[k];
        for i in 1..v.len() { dot += v[i] * b_copy[k + i]; }
        dot *= tau[k];
        b_copy[k] -= dot;
        for i in 1..v.len() { b_copy[k + i] -= dot * v[i]; }
    }

    // Back-substitution: solve R x = b_copy[0..n]
    let mut x = vec![0.0f64; n];
    for i in (0..n).rev() {
        let mut sum = b_copy[i];
        for j in (i+1)..n { sum -= a_copy[i * n + j] * x[j]; }
        x[i] = sum / a_copy[i * n + i];
    }
    x
}
```

## SVD: The Nuclear Option

For severely ill-conditioned or rank-deficient problems, the Singular Value Decomposition is the gold standard:

```
A = U Σ Vᵀ
```

Where U (m×m) and V (n×n) are orthogonal, and Σ (m×n) is diagonal (singular values σ₁ ≥ σ₂ ≥ ... ≥ σᵣ > 0). The least squares solution is:

```
x = V Σ⁺ Uᵀ b
```

Where Σ⁺ = diag(1/σ₁, ..., 1/σᵣ, 0, ..., 0). This handles rank deficiency gracefully (zero singular values → zero contribution). For near-singular problems, truncate small singular values (e.g., set σᵢ = 0 if σᵢ < ε·σ₁). This is the **truncated SVD** or **Tikhonov regularization**.

SVD is O(mn²) — more expensive than QR (also O(mn²) but higher constant). Use it when:
- A is known to be ill-conditioned.
- You want rank information (how many effective degrees of freedom).
- You're doing PCA (principal components = right singular vectors).
- QR gives suspicious results (check ‖Ax - b‖ vs. expected noise level).

## When to Use What

| Problem | Method | Stability | Cost |
|---------|--------|-----------|------|
| Well-conditioned, m ≫ n | QR (Householder) | Excellent | O(mn²) |
| Well-conditioned, m ≥ n, cheap | Normal equations | Poor (squares κ) | O(mn² + n³/3) |
| Mildly ill-conditioned | QR with column pivoting | Very good | O(mn²) |
| Severely ill-conditioned | SVD | Gold standard | O(mn²) ×3–5 of QR |
| Very large, sparse | LSQR (Krylov method) | Good | O(k · nnz) |

In practice: always use QR (LAPACK's `dgels`). If the residual is suspiciously large, try SVD (`dgelss`). Never use the normal equations.

## Benchmark Summary (2000 × 1000 least squares)

| Method | Time | Relative error |
|--------|------|---------------|
| Normal equations (Cholesky) | 18 ms | 1.2 × 10⁻⁶ (good matrix) |
| Normal equations | 18 ms | 3.4 × 10⁰ (κ=10⁸) — garbage |
| QR (LAPACK dgels) | 42 ms | 5.6 × 10⁻¹² (κ=10⁸) |
| SVD (LAPACK dgelss) | 150 ms | 4.2 × 10⁻¹² (κ=10⁸) |

Both QR and SVD deliver correct answers for ill-conditioned matrices. QR is 3.5× faster than SVD. Normal equations are 2.3× faster than QR but produce garbage for ill-conditioned problems. The lesson: correctness before speed. Use QR.
