# Quasi-Newton Methods

Newton's method converges quadratically but requires the n×n Hessian matrix and its inverse — O(n³) per iteration, infeasible for n > 10⁴. Quasi-Newton methods (BFGS, L-BFGS) approximate the Hessian inverse using only gradient information, achieving superlinear convergence at O(n²) per iteration. For n up to ~10⁶, L-BFGS is the gold standard.

## Newton's Method

```
x_{k+1} = x_k - H⁻¹ ∇f(x_k)
```

Where H is the Hessian (n×n matrix of second derivatives). Quadratic convergence: near the optimum, the error squares each iteration. But computing H and solving Hd = -∇f costs O(n³) per iteration.

## BFGS (Broyden–Fletcher–Goldfarb–Shanno)

BFGS builds an approximation Bₖ ≈ H of the Hessian inverse, updating it with each gradient evaluation using the **secant equation**:

```
B_{k+1} = (I - ρₖ sₖ yₖᵀ) Bₖ (I - ρₖ yₖ sₖᵀ) + ρₖ sₖ sₖᵀ
```

Where sₖ = x_{k+1} - xₖ, yₖ = ∇f(x_{k+1}) - ∇f(xₖ), ρₖ = 1 / (yₖᵀ sₖ).

```rust
fn bfgs<F, G>(f: F, grad: G, mut x: Vec<f64>, max_iter: usize,
              tol: f64) -> Vec<f64>
where F: Fn(&[f64]) -> f64,
      G: Fn(&[f64]) -> Vec<f64> {
    let n = x.len();
    let mut b = vec![0.0f64; n * n]; // approximate Hessian inverse
    for i in 0..n { b[i * n + i] = 1.0; } // B₀ = I

    let mut g = grad(&x);
    let mut g_norm = g.iter().map(|&gi| gi * gi).sum::<f64>().sqrt();

    for _ in 0..max_iter {
        if g_norm < tol { break; }

        // Compute search direction: d = -B * g
        let mut d = vec![0.0f64; n];
        for i in 0..n {
            let mut sum = 0.0;
            for j in 0..n { sum += b[i * n + j] * g[j]; }
            d[i] = -sum;
        }

        // Line search to find step size α
        let alpha = backtracking_line_search(&f, &g, &x, &d);

        // Update x
        let mut x_new = x.clone();
        for i in 0..n { x_new[i] += alpha * d[i]; }

        let g_new = grad(&x_new);
        let mut s = vec![0.0f64; n]; // s = x_{k+1} - x_k
        let mut y = vec![0.0f64; n]; // y = g_{k+1} - g_k
        for i in 0..n {
            s[i] = x_new[i] - x[i];
            y[i] = g_new[i] - g[i];
        }

        // ρ = 1 / (yᵀ s)
        let ys: f64 = y.iter().zip(s.iter()).map(|(&yi, &si)| yi * si).sum();
        let rho = if ys > 1e-12 { 1.0 / ys } else { 0.0 };

        if rho > 0.0 {
            // B = (I - ρ s yᵀ) B (I - ρ y sᵀ) + ρ s sᵀ
            // This is O(n²) for the matrix operations
            update_bfgs(&mut b, &s, &y, rho, n);
        }

        x = x_new;
        g = g_new;
        g_norm = g.iter().map(|&gi| gi * gi).sum::<f64>().sqrt();
    }
    x
}
```

BFGS stores and updates an n×n dense matrix. For n = 10,000, that's 800 MB — it fits in RAM but chokes caches. For n = 100,000, it's 80 GB — not feasible.

## L-BFGS (Limited-Memory BFGS)

L-BFGS stores only the last m (typically 3–20) pairs of (sₖ, yₖ) and computes B∇f implicitly via a two-loop recursion — O(mn) instead of O(n²):

```rust
struct LBFGS {
    m: usize,                  // history size
    s_list: Vec<Vec<f64>>,     // last m s vectors
    y_list: Vec<Vec<f64>>,     // last m y vectors
    rho_list: Vec<f64>,        // last m ρ values
}

impl LBFGS {
    fn compute_direction(&self, g: &[f64]) -> Vec<f64> {
        let n = g.len();
        let k = self.s_list.len();
        if k == 0 { return g.iter().map(|&gi| -gi).collect(); }

        let mut q = g.to_vec();
        let mut alpha = vec![0.0f64; k];

        // First loop (forward through history)
        for i in (0..k).rev() {
            let si = &self.s_list[i];
            let yi = &self.y_list[i];
            alpha[i] = self.rho_list[i] * si.iter()
                .zip(q.iter()).map(|(&s, &q)| s * q).sum::<f64>();
            for j in 0..n { q[j] -= alpha[i] * yi[j]; }
        }

        // Initial Hessian approximation: γ = sₖ₋₁ᵀ yₖ₋₁ / yₖ₋₁ᵀ yₖ₋₁
        let gamma = {
            let sk = self.s_list.last().unwrap();
            let yk = self.y_list.last().unwrap();
            let ys = sk.iter().zip(yk.iter())
                .map(|(&s, &y)| s * y).sum::<f64>();
            let yy = yk.iter().map(|&y| y * y).sum::<f64>();
            if yy > 1e-12 { ys / yy } else { 1.0 }
        };

        let mut r: Vec<f64> = q.iter().map(|&qi| gamma * qi).collect();

        // Second loop (backward through history)
        for i in 0..k {
            let si = &self.s_list[i];
            let yi = &self.y_list[i];
            let beta = self.rho_list[i] * yi.iter()
                .zip(r.iter()).map(|(&y, &r)| y * r).sum::<f64>();
            for j in 0..n { r[j] += si[j] * (alpha[i] - beta); }
        }

        // Return -r (the search direction d = -B∇f)
        r.iter().map(|&ri| -ri).collect()
    }
}
```

The two-loop recursion is O(mn) operations. For m = 10 and n = 10⁶: 10⁷ operations per iteration — milliseconds. The memory cost is 2mn = 20 × n = 20 MB for n = 10⁶.

L-BFGS achieves superlinear convergence (nearly as fast as BFGS) while using O(mn) memory and O(mn) time per iteration. This is what makes it the default optimizer for logistic regression, maximum likelihood estimation, and any smooth optimization with n > 1000.

## Backtracking Line Search

Both BFGS and L-BFGS need a step size α that satisfies the **Armijo condition** (sufficient decrease):

```rust
fn backtracking_line_search<F>(f: &F, g: &[f64], x: &[f64],
                                d: &[f64]) -> f64
where F: Fn(&[f64]) -> f64 {
    let mut alpha = 1.0f64;
    let c = 1e-4f64;   // Armijo constant
    let tau = 0.5f64;  // backtracking factor

    let f0 = f(x);
    let gd = g.iter().zip(d.iter())
        .map(|(&gi, &di)| gi * di).sum::<f64>(); // directional derivative

    let mut x_trial = x.to_vec();

    for _ in 0..20 {
        for i in 0..x.len() { x_trial[i] = x[i] + alpha * d[i]; }
        let f_trial = f(&x_trial);
        if f_trial <= f0 + c * alpha * gd { break; }
        alpha *= tau;
    }
    alpha
}
```

The line search evaluates f(x + αd) up to 20 times per BFGS iteration. For expensive functions (e.g., log-likelihood on large datasets), this dominates the runtime. The **Wolfe conditions** add a curvature condition to avoid unnecessarily small steps, but in practice Armijo backtracking is sufficient for most problems.

## When BFGS/L-BFGS Fails

- **Non-differentiable objectives**: BFGS assumes smoothness. Use subgradient methods or ADMM.
- **Stochastic objectives**: BFGS needs accurate gradients. For noisy gradients, use stochastic quasi-Newton (SQN, oLBFGS).
- **Non-convex with many local minima**: BFGS converges to the nearest local minimum. Combine with random restarts or use global optimization methods.
- **Constraints**: Use projected BFGS or interior-point methods (next article).

## Benchmark Summary (Rosenbrock, d = 1000)

| Method | Iterations | Time per iter | Total |
|--------|-----------|--------------|-------|
| Gradient descent + momentum | 3200 | 8 μs | 26 ms |
| Adam | 1800 | 25 μs | 45 ms |
| BFGS | 28 | 1.2 ms | 34 ms |
| L-BFGS (m=10) | 38 | 80 μs | 3.0 ms |
| Newton (exact) | 12 | 8.3 ms (O(n³)) | 100 ms |

L-BFGS is 8.7× faster than gradient descent and 15× faster than Adam for this smooth, ill-conditioned problem. For n > 10⁵, L-BFGS widens the gap because its per-iteration cost grows as O(mn), not O(n²) like BFGS.
