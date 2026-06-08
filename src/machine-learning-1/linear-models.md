# Linear Models

Linear models are the simplest machine learning algorithms: predict a weighted sum of features. Despite their simplicity, they're the first thing to try on any new problem. Logistic regression with proper feature engineering often matches a small neural network — and trains in milliseconds instead of minutes.

## Logistic Regression

The logistic regression case study (Chapter 11) covered inference optimization. Here we focus on **training**: finding the weights w that maximize the likelihood of the training data.

For binary classification with labels y ∈ {0, 1}:

```
minimize_w  -Σ [y_i log σ(wᵀx_i) + (1-y_i) log(1 - σ(wᵀx_i))] + λ‖w‖²
```

Where σ(z) = 1/(1 + e^{-z}) is the sigmoid. This is convex and smooth — it can be solved with gradient descent, L-BFGS, or Newton's method.

```rust
fn logistic_loss(w: &[f64], x: &[f64], y: &[f64], n: usize, d: usize,
                 lambda: f64) -> f64 {
    let mut loss = 0.0f64;
    for i in 0..n {
        let mut dot = 0.0;
        for j in 0..d { dot += w[j] * x[i * d + j]; }
        let sigmoid = 1.0 / (1.0 + (-dot).exp());
        loss += if y[i] > 0.5 {
            -sigmoid.ln()
        } else {
            -(1.0 - sigmoid).ln()
        };
    }
    loss / n as f64 + lambda * w.iter().map(|&wj| wj * wj).sum::<f64>()
}

fn logistic_gradient(w: &[f64], x: &[f64], y: &[f64], n: usize, d: usize,
                     lambda: f64) -> Vec<f64> {
    let mut grad = vec![0.0f64; d];
    for i in 0..n {
        let mut dot = 0.0;
        for j in 0..d { dot += w[j] * x[i * d + j]; }
        let sigmoid = 1.0 / (1.0 + (-dot).exp());
        let error = sigmoid - y[i];
        for j in 0..d { grad[j] += error * x[i * d + j]; }
    }
    for j in 0..d {
        grad[j] = grad[j] / n as f64 + 2.0 * lambda * w[j];
    }
    grad
}
```

The gradient computation is O(nd) — two matrix-vector products (Xw, then Xᵀerror). This is exactly a BLAS-2 operation. For n = 10⁶, d = 100: ~2 ms per gradient evaluation on Zen 2.

Training with L-BFGS converges in ~30 iterations: ~60 ms total. With SGD (mini-batch 256): ~200 iterations × 0.05 ms = 10 ms total. SGD is faster per-epoch but needs more iterations to achieve the same accuracy because of gradient noise.

## SVM: The Dual Formulation

Support Vector Machines solve:

```
minimize_w  ½‖w‖² + C Σ max(0, 1 - y_i (wᵀx_i + b))
```

The **hinge loss** max(0, 1 - y·(wᵀx + b)) is non-smooth — gradient descent struggles. The standard approach solves the **dual**:

```
maximize_α  Σ α_i - ½ Σ Σ α_i α_j y_i y_j K(x_i, x_j)
subject to  0 ≤ α_i ≤ C, Σ α_i y_i = 0
```

Where K(x_i, x_j) is a kernel function (or just the dot product x_iᵀx_j for linear SVM). This is a quadratic program in n variables — O(n²) memory for the Gram matrix. For n > 10⁴, the Gram matrix doesn't fit in memory.

**SMO (Sequential Minimal Optimization)** solves the dual by optimizing two α's at a time (the smallest subproblem that respects the constraint Σ α_i y_i = 0). Each step is O(1) after precomputing the kernel matrix — or O(n) if computing kernels on the fly. LibSVM and scikit-learn's SVM use SMO.

## Primal SGD for Linear SVM

For the linear case (no kernel), SGD on the primal is O(d) per sample and O(n) per epoch — much cheaper than the dual:

```rust
fn svm_sgd(x: &[f64], y: &[f64], n: usize, d: usize,
           c: f64, learning_rate: f64, epochs: usize) -> Vec<f64> {
    let mut w = vec![0.0f64; d];
    let mut b = 0.0f64;

    for epoch in 0..epochs {
        let lr = learning_rate / (1.0 + epoch as f64);
        for i in 0..n {
            let mut dot = b;
            for j in 0..d { dot += w[j] * x[i * d + j]; }
            let margin = y[i] * dot;
            if margin < 1.0 {
                // Misclassified or within margin: subgradient step
                for j in 0..d {
                    w[j] = w[j] - lr * (-c * y[i] * x[i * d + j] + w[j] / n as f64);
                }
                b -= lr * (-c * y[i]);
            } else {
                // Correctly classified: only regularization
                for j in 0..d { w[j] -= lr * w[j] / n as f64; }
            }
        }
    }
    w
}
```

SGD on the primal solves linear SVM in O(n·d·epochs) time with O(d) memory. For n = 10⁶, d = 100, and 10 epochs: ~1 second. The dual SMO would need 10¹² entries in the Gram matrix — impossible. Always use the primal for linear SVMs on large datasets.

## Regularization and the Lasso

Ridge regression (ℓ₂ penalty): shrinks coefficients toward zero, differentiable, easy with SGD/L-BFGS.

Lasso (ℓ₁ penalty): produces sparse solutions (some coefficients become exactly zero). The objective is non-differentiable at zero — use **coordinate descent** or **proximal gradient**:

```rust
fn lasso_coordinate_descent(x: &[f64], y: &[f64], n: usize, d: usize,
                             lambda: f64, max_iter: usize) -> Vec<f64> {
    let mut w = vec![0.0f64; d];
    let mut residuals = y.to_vec();

    for _ in 0..max_iter {
        for j in 0..d {
            // Compute how much changing w[j] affects the loss
            let mut grad_j = 0.0;
            let mut xj_norm_sq = 0.0;
            for i in 0..n {
                grad_j += x[i * d + j] * residuals[i];
                xj_norm_sq += x[i * d + j] * x[i * d + j];
            }
            grad_j = -grad_j / n as f64 + w[j] * xj_norm_sq / n as f64;
            xj_norm_sq /= n as f64;

            let w_old = w[j];
            // Soft thresholding (proximal operator for ℓ₁)
            w[j] = (w_old - grad_j / xj_norm_sq).abs()
                .max(0.0)
                .copysign(w_old - grad_j / xj_norm_sq)
                * (1.0 - lambda / ((w_old - grad_j / xj_norm_sq).abs() * xj_norm_sq))
                .max(0.0);

            let delta = w[j] - w_old;
            if delta.abs() > 1e-12 {
                for i in 0..n {
                    residuals[i] -= delta * x[i * d + j];
                }
            }
        }
    }
    w
}
```

Coordinate descent for the Lasso converges in ~30 passes over the features for typical λ. Each pass costs O(nd). Total: O(nd × 30) — comparable to L-BFGS for ridge regression.

## When to Use What

| Problem | Algorithm | Training cost |
|---------|-----------|--------------|
| Binary classification, n > d | Logistic regression + L-BFGS | O(nd × 30) |
| Binary classification, n ≫ d | Logistic regression + SGD | O(nd × 10) |
| Multi-class classification | Multinomial logistic + SGD | O(nd·K × 10) |
| Linear SVM, large n | Primal SGD | O(nd × 10) |
| Kernel SVM, small n (< 10⁴) | SMO (dual) | O(n²) |
| Sparse linear model | Lasso (coordinate descent) | O(nd × 30) |
| Elastic Net (ℓ₁+ℓ₂) | Coordinate descent | O(nd × 30) |

The logistic regression case study (Chapter 11) showed that inference can be optimized to ~0.1 μs per sample with SIMD. Training is 3–4 orders of magnitude more expensive — but applying the same SIMD and blocking techniques to the gradient computation gives the same proportional speedups.
