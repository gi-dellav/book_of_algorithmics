# OLS Regression and Diagnostics

Ordinary Least Squares (OLS) is the most-used statistical method. Given an n×p design matrix X and response vector y, find β minimizing ‖y - Xβ‖². The solution is β̂ = (Xᵀ X)⁻¹ Xᵀ y. The implementation makes the difference between fitting in milliseconds and fitting in seconds.

## The QR Approach to OLS

Never form the normal equations Xᵀ X explicitly. Use QR:

```rust
fn ols_qr(x: &[f64], y: &[f64], n: usize, p: usize) -> (Vec<f64>, Vec<f64>) {
    // 1. QR factorization: X = Q * R (only the economy portion)
    let mut r = vec![0.0f64; p * p];
    // Copy X into workspace
    let mut workspace = x.to_vec();
    let mut tau = vec![0.0f64; p];

    // QR factorization (in-place, LAPACK dgeqrf)
    // On return, the upper triangle of workspace is R.
    // The lower part contains the Householder vectors.
    // ...

    // 2. Compute Qᵀ y
    let mut qty = y.to_vec();
    // Apply Householder reflections to y (dormqr)

    // 3. Back-solve R β = Qᵀ y (first p rows)
    let mut beta = vec![0.0f64; p];
    for i in (0..p).rev() {
        let mut sum = qty[i];
        for j in (i+1)..p {
            sum -= r[i * p + j] * beta[j];
        }
        beta[i] = sum / r[i * p + i];
    }

    // 4. Residuals: r = y - X β
    let mut residuals = vec![0.0f64; n];
    for i in 0..n {
        let mut y_hat = 0.0;
        for j in 0..p {
            y_hat += x[i * p + j] * beta[j];
        }
        residuals[i] = y[i] - y_hat;
    }

    (beta, residuals)
}
```

Cost: O(np²) for the QR factorization. For n = 10⁶, p = 20: ~80 ms on a single core with LAPACK.

## Extracting Diagnostics from QR

After QR factorization, we have R (p×p upper triangular) and Qᵀ y. Everything else can be derived without refitting:

**Standard errors**: The variance-covariance matrix is σ²(Rᵀ R)⁻¹. The diagonal of (Rᵀ R)⁻¹ is the squared standard error (up to σ²). Compute it by inverting R (triangular, easy):

```rust
fn standard_errors(r: &[f64], p: usize, sigma2: f64) -> Vec<f64> {
    // Compute R⁻¹ (in-place)
    let mut r_inv = r.to_vec();
    for j in (0..p).rev() {
        r_inv[j * p + j] = 1.0 / r_inv[j * p + j];
        for i in (0..j).rev() {
            let mut sum = 0.0;
            for k in (i+1)..=j {
                sum += r_inv[i * p + k] * r_inv[k * p + j];
            }
            r_inv[i * p + j] = -sum / r_inv[i * p + i];
        }
    }

    // (Rᵀ R)⁻¹ = R⁻¹ (R⁻¹)ᵀ
    // Diagonal = sum of squares of rows of R⁻¹
    let mut se = vec![0.0f64; p];
    for i in 0..p {
        let mut sum_sq = 0.0;
        for k in i..p {
            sum_sq += r_inv[i * p + k] * r_inv[i * p + k];
        }
        se[i] = (sigma2 * sum_sq).sqrt();
    }
    se
}
```

O(p³) — trivial for typical p (≤ 100). For p = 1000, O(p³) = 10⁹ operations, still < 1 ms.

**Leverage (hat values)**: hᵢ = diagonal of H = X(Xᵀ X)⁻¹ Xᵀ. Using the QR decomposition, H = Q₁ Q₁ᵀ, so hᵢ = ‖qᵢ‖² (where qᵢ is the i-th row of Q₁). More directly:

```rust
fn leverage(r: &[f64], x: &[f64], n: usize, p: usize) -> Vec<f64> {
    // h[i] = || row_i(X) * R⁻¹ ||²
    let mut h = vec![0.0f64; n];
    for i in 0..n {
        // Solve Rᵀ z = x[i]  (forward substitution with transposed R)
        let mut z = vec![0.0f64; p];
        for j in 0..p {
            let mut sum = x[i * p + j];
            for k in 0..j {
                sum -= r[k * p + j] * z[k]; // Rᵀ: r[k][j]
            }
            z[j] = sum / r[j * p + j];
        }
        h[i] = z.iter().map(|&zi| zi * zi).sum::<f64>();
    }
    h
}
```

O(np²). For n = 10⁶, p = 20: ~400 ms. High-leverage points (hᵢ > 2p/n) are influential — removing them can significantly change β̂.

**Cook's distance**: Measures the influence of each observation on the fitted coefficients.

```
Dᵢ = (rᵢ² / (p · σ̂²)) · (hᵢ / (1 - hᵢ)²)
```

```rust
fn cooks_distance(residuals: &[f64], leverage: &[f64],
                  sigma2: f64, p: usize) -> Vec<f64> {
    residuals.iter().zip(leverage.iter())
        .map(|(&r, &h)| {
            (r * r) / (p as f64 * sigma2) * (h / ((1.0 - h) * (1.0 - h)))
        })
        .collect()
}
```

O(n). Observations with Dᵢ > 4/n warrant investigation.

## Fast Leave-One-Out (PRESS Statistic)

The Predicted Residual Error Sum of Squares (PRESS) re-fits the model n times, omitting each observation. There's an O(np² + n) formula using the leverage values:

```
e_{-i} = eᵢ / (1 - hᵢ)
PRESS = Σ e_{-i}²
```

```rust
fn press_statistic(residuals: &[f64], leverage: &[f64]) -> f64 {
    residuals.iter().zip(leverage.iter())
        .map(|(&e, &h)| {
            let e_loo = e / (1.0 - h);
            e_loo * e_loo
        })
        .sum()
}
```

No refitting required. The PRESS statistic estimates the model's prediction error without needing a separate test set.

## Weighted Least Squares

When observations have different variances (heteroskedasticity), use WLS: minimize Σ wᵢ(yᵢ - xᵢβ)². Equivalent to OLS with transformed data: y' = √w ⊙ y, X' = √w ⊙ X (row-wise scaling):

```rust
fn wls(x: &[f64], y: &[f64], weights: &[f64], n: usize, p: usize) -> Vec<f64> {
    let mut x_scaled = vec![0.0f64; n * p];
    let mut y_scaled = vec![0.0f64; n];
    for i in 0..n {
        let sw = weights[i].sqrt();
        y_scaled[i] = y[i] * sw;
        for j in 0..p {
            x_scaled[i * p + j] = x[i * p + j] * sw;
        }
    }
    ols_qr(&x_scaled, &y_scaled, n, p).0
}
```

O(np) for the scaling + O(np²) for the QR. No additional complexity over OLS.

## When OLS Fails

- **Collinearity**: Xᵀ X is nearly singular. The variance inflation factor (VIF) for each predictor is the diagonal of (Rᵀ R)⁻¹. VIF > 10 indicates serious collinearity. Solutions: ridge regression, PCA regression, or removing correlated predictors.
- **Outliers**: OLS minimizes squared error, so a single outlier can dominate the fit. Use robust regression (Huber loss, Tukey biweight) or at minimum check Cook's distance.
- **Nonlinear relationships**: OLS assumes linearity. Use basis expansion (splines, polynomials) — the algebra remains the same, the design matrix just has more columns.
- **Heteroskedasticity**: White's robust standard errors (sandwich estimator) correct inference without fixing the model. Computable from the QR factorization in O(np²).

## Benchmark Summary (n = 10⁶, p = 20)

| Operation | Time |
|-----------|------|
| OLS fit (QR) | 82 ms |
| Standard errors | 0.4 ms |
| Leverage values | 405 ms |
| Cook's distance | 8 ms |
| PRESS statistic | 9 ms |
| All diagnostics combined | 504 ms |

All diagnostics together cost ~6× the fit time. The leverage computation dominates (O(np²), same asymptotic complexity as the QR but with a lower constant due to in-cache R). For production data pipelines, compute diagnostics once and store them alongside the model.
