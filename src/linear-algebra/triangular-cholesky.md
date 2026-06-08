# Triangular Systems and Cholesky

Triangular systems are the simplest linear systems: solve Lx = b where L is lower triangular. The solution is O(n²) — cheap compared to O(n³) factorizations. But the naive algorithm is scalar and sequential. Blocked algorithms turn triangular solve into a BLAS-3 operation, achieving ~60% of peak GFLOPS instead of ~5%.

## Forward Substitution (Scalar)

Solve Lx = b where L is lower triangular (L[i][j] = 0 for j > i):

```
x[0] = b[0] / L[0][0]
for i in 1..n:
    sum = 0
    for j in 0..i:
        sum += L[i][j] * x[j]
    x[i] = (b[i] - sum) / L[i][i]
```

```rust
fn forward_sub(l: &[f64], b: &[f64], x: &mut [f64], n: usize) {
    for i in 0..n {
        let mut sum = b[i];
        for j in 0..i {
            sum -= l[i * n + j] * x[j];
        }
        x[i] = sum / l[i * n + i];
    }
}
```

The inner loop computes a dot product of row `L[i][0..i]` with `x[0..i]`. This is BLAS-1: O(n) FLOPs per row, O(n) memory per row. For n = 10,000: ~0.5 seconds on Zen 2 (~200 MFLOPS, <1% of peak). The bottleneck: `x[j]` is accessed non-sequentially (random across the row), and the dot product has a dependency chain (the scalar sum).

## Blocked Forward Substitution

Partition L and x into blocks:

```
[ L11   0   ] [ x1 ]   [ b1 ]
[ L21  L22  ] [ x2 ] = [ b2 ]

x1 = L11⁻¹ b1          (solve smaller triangular system)
x2 = L22⁻¹ (b2 - L21 x1)  (matrix-vector multiply + triangular solve)
```

The `L21 * x1` step is a matrix-vector multiply (BLAS-2). For large blocks, we can go further:

```rust
fn blocked_forward_sub(l: &[f64], b: &[f64], x: &mut [f64], n: usize, block_size: usize) {
    for k in (0..n).step_by(block_size) {
        let end = (k + block_size).min(n);
        let nb = end - k;

        // Solve the diagonal block (small triangular system, O(nb²))
        forward_sub(&l[k * n + k..], &b[k..], &mut x[k..], nb, n);

        // Update the right-hand side for the remaining rows (BLAS-2: dgemv)
        if end < n {
            for i in end..n {
                for j in k..end {
                    b_ptr[i] -= l[i * n + j] * x[j];
                }
            }
        }
    }
}
```

For nb = 128, the `dgemv` (matrix-vector multiply) processes 128 × (n-k) elements per block step, doing O(nb × (n-k)) FLOPs on O(nb × (n-k)) memory — still a 2:1 ratio (memory-bound). We need one more level of blocking.

## Recursive Block Triangular Solve

The truly fast approach is recursive blocking. Split the matrix in half:

```
L * x = b

If n is small (≤ 256): use scalar forward substitution.
Otherwise:
    Solve L11 * x1 = b1 (recursive call on top-left block)
    Update: b2 = b2 - L21 * x1 (dgemm: BLAS-3!)
    Solve L22 * x2 = b2 (recursive call on bottom-right block)
```

The `dgemm` step is a BLAS-3 operation: multiply an (n/2)×(n/2) matrix by a vector, which is O(n²/2) FLOPs on O(n²/2 + n) memory — leading dimension O(n) ratio. For n = 10,000, this is compute-bound for the top-level recursion.

```rust
fn rec_trsm(l: &[f64], b: &mut [f64], n: usize, ld: usize) {
    if n <= 256 {
        // Base case: scalar forward substitution
        forward_sub(l, b, n, ld);
        return;
    }
    let n1 = n / 2;
    let n2 = n - n1;

    // Solve L11 * x1 = b1
    rec_trsm(&l[0], &mut b[0], n1, ld);

    // Update b2 = b2 - L21 * x1   (dgemm)
    unsafe {
        dgemm_(
            b"N", b"N",
            &(n2 as i32), &1i32, &(n1 as i32),
            &(-1.0f64),
            &l[n1 * ld], &(ld as i32),
            &b[0], &(n1 as i32),
            &1.0f64,
            &mut b[n1], &(n2 as i32),
        );
    }

    // Solve L22 * x2 = b2
    rec_trsm(&l[n1 * ld + n1], &mut b[n1], n2, ld);
}
```

For n = 10,000: ~14 ms (7.1 GFLOPS, 22% of peak). Still below dgemm peak because the base case (n ≤ 256) is still memory-bound, and there are many small recursive calls. But it's 35× faster than scalar forward substitution.

## Cholesky Factorization

For a Symmetric Positive Definite (SPD) matrix A, Cholesky factorization computes A = LLᵀ where L is lower triangular. It's cheaper than LU (no pivoting needed for SPD matrices) and numerically stable:

```rust
fn cholesky_scalar(a: &mut [f64], n: usize) {
    for j in 0..n {
        // Diagonal element
        let mut sum = a[j * n + j];
        for k in 0..j {
            let l_jk = a[j * n + k];
            sum -= l_jk * l_jk;
        }
        a[j * n + j] = sum.sqrt();

        // Column below diagonal
        for i in (j+1)..n {
            let mut sum = a[i * n + j];
            for k in 0..j {
                sum -= a[i * n + k] * a[j * n + k];
            }
            a[i * n + j] = sum / a[j * n + j];
        }
    }
}
```

O(n³/3) operations. The scalar version gets ~800 MFLOPS on Zen 2 — about 2.5% of peak. The inner loops are dot products (sum of products), which are memory-bound.

The blocked version in LAPACK (`dpotrf`) uses the recursive pattern described above and achieves ~26 GFLOPS (81% of peak) for n = 4000.

## Detecting Positive Definiteness

If A is not positive definite, the Cholesky factorization will fail (sqrt of negative number, or division by zero if a pivot is zero). In floating point, near-zero pivots cause catastrophic cancellation. LAPACK's `dpotrf` returns an error code if the matrix is not numerically positive definite.

If you don't know whether your matrix is SPD, use LDLᵀ factorization (`dsytrf`) or LU with partial pivoting (`dgetrf`). The performance difference (1/3 n³ vs. 2/3 n³) is less important than correctness.

## When to Use What

| Problem | Use | Notes |
|---------|-----|-------|
| `Ax = b`, A is general | LU (dgetrf + dgetrs) | O((2/3)n³ + O(n²)) |
| `Ax = b`, A is SPD | Cholesky (dpotrf + dpotrs) | O((1/3)n³ + O(n²)) |
| `Ax = b`, A is symmetric | LDLᵀ (dsytrf + dsytrs) | O((1/3)n³ + O(n²)) |
| `Ax = b`, A is triangular | Triangular solve (dtrsm) | O(n²) — just solve, don't factorize |
| `Ax = b` for many b | Factorize once, solve many times (dgetrs) | Factorization is the expensive part |
| `A⁻¹` needed explicitly | Don't. Solve AX = I instead | Matrix inversion is numerically unstable |

The last point is worth emphasizing: you almost never need an explicit matrix inverse. If you think you do, you probably want `factorize(A)` then `solve(A, b)` for each right-hand side b. Computing A⁻¹ explicitly costs 2–3× more than factorization and loses precision.
