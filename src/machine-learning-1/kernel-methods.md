# Kernel Methods and SVMs

Kernel methods let linear models operate in infinite-dimensional feature spaces — without ever computing the features. The **kernel trick** replaces dot products xᵢᵀxⱼ with K(xᵢ, xⱼ), which implicitly computes the dot product in a high-dimensional (possibly infinite-dimensional) feature space. The price: the Gram matrix K of size n×n.

## The Kernel Trick

A kernel K(x, z) = ⟨φ(x), φ(z)⟩ for some feature map φ. Common kernels:

- **Linear**: K(x, z) = xᵀz (no feature map, just the dot product)
- **Polynomial**: K(x, z) = (γ xᵀz + r)^d (feature space of all monomials up to degree d)
- **RBF (Gaussian)**: K(x, z) = exp(-γ‖x - z‖²) (infinite-dimensional feature space)
- **Sigmoid**: K(x, z) = tanh(γ xᵀz + r)

The RBF kernel is the most popular — it's a universal approximator (can represent any continuous function on a compact set, given enough data).

## The Kernel SVM Problem

```
maximize_α  Σα_i - ½ Σᵢⱼ α_i α_j y_i y_j K(x_i, x_j)
subject to  0 ≤ α_i ≤ C,  Σ α_i y_i = 0
```

The Gram matrix K_ij = K(x_i, x_j) is n×n. Computing it is O(n²d) for most kernels. Storing it is O(n²). For n = 100,000, that's 10¹⁰ entries × 8 bytes = 80 GB — doesn't fit in RAM.

## Nyström Approximation

The Nyström method approximates the full Gram matrix K using a subset of m ≪ n **landmark points**:

```
K ≈ K_{nm} · K_{mm}⁻¹ · K_{mn}
```

Where K_{nm} is the n×m kernel matrix between all points and the landmarks, and K_{mm} is the m×m kernel matrix between landmarks.

```rust
fn nystrom_approximation(x: &[f64], n: usize, d: usize, m: usize,
                         gamma: f64) -> (Vec<f64>, Vec<f64>) {
    // Sample m landmark points (random or k-means centroids)
    let landmarks: Vec<usize> = (0..m).map(|_| rand::random::<usize>() % n).collect();

    // Compute K_mm (m×m kernel matrix of landmarks)
    let mut k_mm = vec![0.0f64; m * m];
    for i in 0..m {
        for j in 0..m {
            k_mm[i * m + j] = rbf_kernel(
                &x[landmarks[i] * d..], &x[landmarks[j] * d..], d, gamma,
            );
        }
    }

    // Compute K_mm^(-1/2) via eigendecomposition
    // K_mm = U Λ Uᵀ, then K_mm^(-1/2) = U Λ^(-1/2) Uᵀ
    let (eigvals, eigvecs) = symmetric_eigen(&k_mm, m);

    // Feature map: φ_Nyström(x) = Λ^(-1/2) Uᵀ [K(x, x₁), ..., K(x, x_m)]ᵀ
    // This maps any point to an m-dimensional feature space
    let mut feature_map = vec![0.0f64; m * m];
    for i in 0..m {
        let scale = 1.0 / eigvals[i].sqrt();
        for j in 0..m {
            feature_map[j * m + i] = eigvecs[i * m + j] * scale;
        }
    }

    // Transform all data points using this feature map
    let mut x_transformed = vec![0.0f64; n * m];
    for i in 0..n {
        for j in 0..m {
            let k_ij = rbf_kernel(
                &x[i * d..], &x[landmarks[j] * d..], d, gamma,
            );
            for k in 0..m {
                x_transformed[i * m + k] += k_ij * feature_map[j * m + k];
            }
        }
    }

    (x_transformed, landmarks.iter().map(|&l| l as f64).collect())
}

fn rbf_kernel(x: &[f64], z: &[f64], d: usize, gamma: f64) -> f64 {
    let sq_dist: f64 = x.iter().zip(z.iter())
        .map(|(&xi, &zi)| (xi - zi) * (xi - zi))
        .sum();
    (-gamma * sq_dist).exp()
}
```

The Nyström approximation reduces the SVM training cost from O(n²) to O(n·m² + m³). For m = 1000 and n = 10⁶, training takes O(10⁶ × 10³) = 10⁹ operations — seconds instead of hours.

## Random Fourier Features (RFF)

For the RBF kernel specifically, Random Fourier Features provide an even simpler approximation. The RBF kernel is the Fourier transform of a Gaussian distribution:

```
K(x, z) = E_{ω ~ N(0, 2γI)} [cos(ωᵀx + b) · cos(ωᵀz + b)]
```

Draw D random frequencies ω_d ~ N(0, 2γI) and random offsets b_d ~ Uniform(0, 2π):

```rust
fn random_fourier_features(x: &[f64], n: usize, d: usize, d_rff: usize,
                            gamma: f64) -> Vec<f64> {
    let mut rng = rand::thread_rng();
    let mut x_rff = vec![0.0f64; n * d_rff];

    // Generate random frequencies and offsets
    let sigma = (2.0 * gamma).sqrt();
    let mut omega = vec![0.0f64; d_rff * d];
    let mut bias = vec![0.0f64; d_rff];

    for j in 0..d_rff {
        for k in 0..d {
            omega[j * d + k] = rng.sample(rand_distr::StandardNormal) * sigma;
        }
        bias[j] = rng.gen_range(0.0..2.0 * std::f64::consts::PI);
    }

    // Transform: φ_j(x) = √(2/D) · cos(ω_jᵀx + b_j)
    let scale = (2.0 / d_rff as f64).sqrt();
    for i in 0..n {
        for j in 0..d_rff {
            let mut dot = bias[j];
            for k in 0..d {
                dot += omega[j * d + k] * x[i * d + k];
            }
            x_rff[i * d_rff + j] = scale * dot.cos();
        }
    }
    x_rff
}
```

RFF maps n points from d dimensions to D dimensions (typically D = 1000–5000) in O(ndD) time. After the transformation, a linear SVM (or logistic regression) on the D-dimensional features approximates the kernel SVM. Error: O(1/√D). For D = 2000, the approximation error is < 2% for most problems.

RFF is the go-to method for scaling kernel methods to n > 10⁵. It's used in production at companies that need the theoretical guarantees of kernel methods (reproducing kernel Hilbert spaces, well-calibrated uncertainties) without the O(n²) cost.

## When Kernels Beat Neural Networks

| Property | Kernel SVM | Neural Network |
|----------|-----------|---------------|
| Training on small n (< 10³) | Excellent | Poor (overfits) |
| Training on large n (> 10⁶) | Needs approximation | Excellent |
| Theoretical guarantees | Strong (convex, RKHS) | Weak (non-convex) |
| Uncertainty quantification | Well-calibrated (GP) | Needs Bayesian NN |
| Feature engineering needed | With RBF, minimal | With CNN/transformer, minimal |
| Interpretability | Support vectors explain decisions | Black box |

For n < 10,000, kernel SVM with RBF is often the most accurate classifier available — it's convex (no local minima), it handles nonlinearity naturally (RBF kernel), and it comes with strong generalization guarantees (margin maximization). The only obstacle is the O(n²) scaling — which Nyström and RFF solve.
