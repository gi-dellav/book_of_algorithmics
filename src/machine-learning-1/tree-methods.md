# Tree-Based Methods

For tabular data, gradient boosted trees are still the best off-the-shelf predictor. They handle missing values, mixed data types, and nonlinear relationships automatically. XGBoost and LightGBM dominate Kaggle competitions for a reason — and that reason is computational engineering.

## Decision Trees

A decision tree recursively splits the data to maximize information gain. At each node, find the feature j* and threshold t* that best separate the classes:

```rust
fn find_best_split(x: &[f64], y: &[f64], n: usize, d: usize) -> (usize, f64) {
    let mut best_gain = 0.0f64;
    let mut best_j = 0usize;
    let mut best_t = 0.0f64;

    for j in 0..d {
        // Sort by feature j
        let mut indices: Vec<usize> = (0..n).collect();
        indices.sort_by(|&a, &b| x[a * d + j].partial_cmp(&x[b * d + j]).unwrap());

        // Scan sorted values, update histograms incrementally
        // O(n log n) per feature for sorting, O(n) for scanning
        // ...
    }
    (best_j, best_t)
}
```

Naive splitting costs O(d · n log n) per node, and there are O(n) nodes in the worst case — O(d · n² log n) total. For n = 10⁵, d = 20, this is ~10¹² operations — minutes per tree.

## Histogram-Based Splitting (LightGBM)

LightGBM's key innovation: instead of sorting by each feature, **bucket** continuous values into 256 discrete bins:

```rust
fn histogram_split(x: &[f64], y: &[f64], n: usize, d: usize,
                   num_bins: usize) -> (usize, f64) {
    // Step 1: Build histograms for each feature
    let mut histograms: Vec<Vec<(f64, f64)>> = vec![vec![(0.0, 0.0); num_bins]; d];
    // histogram[j][b] = (sum of gradients, sum of hessians) for feature j, bin b

    // Step 2: Find best split by scanning histograms
    let mut best_gain = -1.0f64;
    let mut best_j = 0usize;
    let mut best_b = 0usize;

    for j in 0..d {
        let mut left_sum = 0.0f64;
        let mut left_hess = 0.0f64;
        let total_sum: f64 = histograms[j].iter().map(|&(s, _)| s).sum();
        let total_hess: f64 = histograms[j].iter().map(|&(_, h)| h).sum();

        for b in 0..num_bins - 1 {
            left_sum += histograms[j][b].0;
            left_hess += histograms[j][b].1;
            let right_sum = total_sum - left_sum;
            let right_hess = total_hess - left_hess;

            let gain = left_sum * left_sum / (left_hess + 1.0)
                     + right_sum * right_sum / (right_hess + 1.0)
                     - total_sum * total_sum / (total_hess + 1.0);
            if gain > best_gain {
                best_gain = gain;
                best_j = j;
                best_b = b;
            }
        }
    }
    (best_j, /* bin_to_value(best_j, best_b) */)
}
```

Histogram construction is O(nd) — scan the data once, incrementing bucket counters. Finding the best split is O(d × num_bins) = O(d × 256) — trivial. Total per level: O(nd). For a tree of depth 8: O(8nd) — linear in n.

This is why LightGBM is 10–100× faster than scikit-learn's `GradientBoostingClassifier`: histogram splitting replaces O(d · n log n) with O(d · n). The loss in accuracy from bucketing is negligible (256 bins is enough for any practical regression/classification problem).

## Gradient Boosting

Gradient boosting trains trees sequentially, each tree fitting the negative gradient (residual) of the loss function:

```rust
fn gradient_boosting(x: &[f64], y: &[f64], n: usize, d: usize,
                     n_trees: usize, learning_rate: f64) -> Vec<Vec<(usize, f64, f64)>> {
    let mut models: Vec<Vec<(usize, f64, f64)>> = Vec::new();
    let mut predictions = vec![0.0f64; n];

    for _ in 0..n_trees {
        // Compute gradients and hessians
        let mut grad = vec![0.0f64; n];
        let mut hess = vec![0.0f64; n];
        for i in 0..n {
            let p = 1.0 / (1.0 + (-predictions[i]).exp());
            grad[i] = p - y[i];
            hess[i] = p * (1.0 - p);
        }

        // Build histogram and find best splits
        let tree = build_histogram_tree(x, &grad, &hess, n, d, /* max_depth */);

        // Update predictions
        for i in 0..n {
            predictions[i] += learning_rate * tree_predict(&tree, x, i, d);
        }
        models.push(tree);
    }
    models
}
```

The computational bottleneck is the gradient/hessian computation (O(nd)) and the histogram construction (O(nd)). Both are streaming operations — the data is accessed sequentially, cache-friendly. LightGBM additionally uses:
- **Exclusive Feature Bundling**: Combine mutually exclusive sparse features into a single feature, reducing d.
- **Gradient-based One-Side Sampling (GOSS)**: Keep all instances with large gradients (hard to fit) and sample from instances with small gradients. Reduces effective n by ~60% without loss in accuracy.

## XGBoost's Approximate Splitting

XGBoost uses weighted quantile sketches to find candidate split points in O(n) per feature without sorting. It maintains a compressed summary of the data distribution and proposes splits at the ϵ-quantiles:

```
For each feature:
  Maintain a weighted quantile sketch (Greenwald-Khanna).
  Propose splits at quantiles {0, 1/k, 2/k, ..., 1}.
  Find best split among the proposed candidates.
```

The sketch is O(n) to build and O(k) to query. Default k = 20 in XGBoost's `approx` mode — 20 candidate splits per feature is enough for most problems.

## GPU Tree Building

LightGBM and XGBoost support GPU-accelerated tree building. The key techniques:

1. **Data layout**: Features stored in column-major format (each feature is a contiguous array). Histogram updates are then `atomicAdd` on GPU global memory.
2. **Histogram reduction**: 256 bins × d features = compact. Fit in shared memory (48 KB per SM).
3. **Multi-tree parallelism**: Build multiple trees simultaneously on different GPU multiprocessors.

On an NVIDIA A100, GPU histogram tree building achieves ~100× speedup over CPU for n = 10⁷. The bottleneck shifts from compute to PCIe bandwidth — the data must fit in GPU memory.

## When to Use What

| Data size | Method | Training time (typical) |
|-----------|--------|------------------------|
| n < 10³ | scikit-learn RandomForest | < 100 ms |
| 10³ < n < 10⁶ | LightGBM (CPU) | Seconds to minutes |
| n > 10⁶ | LightGBM (GPU) | Seconds |
| n > 10⁹ | XGBoost + external memory | Minutes to hours |
| Streaming data | Online boosting (Hoeffding trees) | Per-sample update |

For 90% of tabular data problems, `LightGBM` with default parameters (100 trees, depth 6, 256 bins) gives state-of-the-art results in under a minute. Neural networks on the same tabular data typically require 100× more training time for comparable accuracy.
