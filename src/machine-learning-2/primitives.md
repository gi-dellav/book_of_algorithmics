# Neural Network Primitives

Every neural network is built from a small set of primitives. Each primitive has a forward pass (compute the output) and a backward pass (compute gradients with respect to inputs and parameters). The computational cost of each primitive determines the cost of training. This article catalogs the primitives and their bottlenecks.

## Fully-Connected (Linear) Layer

```
y = Wx + b

Forward: O(d_out × d_in)
Backward: dW = dy · xᵀ, dx = Wᵀ · dy, db = dy    (same O(d_out × d_in))
```

The core operation is matrix multiplication. For a layer with d_in = 1024, d_out = 1024, batch = 64: the forward pass is a 64×1024 × 1024×1024 matmul — 64 × 2 × 1024² ≈ 134M FLOPs. The backward pass is two more matmuls of the same size. Total: 400M FLOPs per layer per batch. On Zen 2 (32 GFLOPS single-core): ~12.5 ms. On an A100 (312 TFLOPS): ~1.3 μs.

## Convolutional Layer

A 2D convolution with filter size K×K, C_in input channels, C_out output channels:

```
y[h][w][c_out] = Σ_{i,j,c_in} W[i][j][c_in][c_out] · x[h+i][w+j][c_in]
```

Naive: O(H·W·K²·C_in·C_out). With im2col + matmul: reshape the input so the convolution becomes a matrix multiply. im2col expands each K×K×C_in patch into a row of size K²·C_in, producing a matrix of shape (H·W) × (K²·C_in). Then multiply by W reshaped to (K²·C_in) × C_out.

```rust
fn im2col(x: &[f32], h: usize, w: usize, c_in: usize,
          k: usize, stride: usize) -> Vec<f32> {
    let h_out = (h - k) / stride + 1;
    let w_out = (w - k) / stride + 1;
    let mut cols = vec![0.0f32; h_out * w_out * k * k * c_in];

    for oh in 0..h_out {
        for ow in 0..w_out {
            for kh in 0..k {
                for kw in 0..k {
                    for ci in 0..c_in {
                        let col_idx = ((oh * w_out + ow) * k * k + kh * k + kw) * c_in + ci;
                        let x_idx = ((oh * stride + kh) * w + (ow * stride + kw)) * c_in + ci;
                        cols[col_idx] = x[x_idx];
                    }
                }
            }
        }
    }
    cols
}
```

im2col expands the input by a factor of K² — for K=3, that's 9× memory. For large images (224×224), this is acceptable. For gigapixel images, use Winograd convolution or FFT convolution to reduce FLOPs.

The backward pass of convolution is itself a convolution (with flipped filters) — same complexity as the forward pass.

## Batch Normalization

```
μ = mean(x)
σ² = var(x)
x̂ = (x - μ) / √(σ² + ε)
y = γ · x̂ + β
```

Forward: O(batch_size × features). Backward: O(batch_size × features) with some extra terms for μ and σ² gradients.

Batch norm stabilizes training (reduces internal covariate shift) but adds ~5% overhead. For inference, μ and σ² are replaced with running averages, and the layer becomes a simple affine transform y = α·x + β (fusable into the preceding linear layer).

## Layer Normalization

Same as batch norm but normalizes over features instead of the batch dimension. Used in transformers because:
1. Batch size can be small (1 for inference).
2. Sequence length varies — batch norm's running statistics are unreliable.

Computational cost is identical to batch norm. The backward pass is slightly simpler (no cross-sample dependencies).

## Self-Attention (Scaled Dot-Product)

The core of transformers:

```
Q = X·W_Q    (n × d_k)
K = X·W_K    (n × d_k)
V = X·W_V    (n × d_v)

Attention(Q, K, V) = softmax(Q·Kᵀ / √d_k) · V
```

```rust
fn attention(q: &[Vec<f32>], k: &[Vec<f32>], v: &[Vec<f32>],
             d_k: f32) -> Vec<Vec<f32>> {
    let n = q.len();
    let d_v = v[0].len();
    let scale = 1.0 / d_k.sqrt();

    // Q·Kᵀ: O(n² · d_k)
    let mut scores = vec![vec![0.0f32; n]; n];
    for i in 0..n {
        for j in 0..n {
            let mut dot = 0.0;
            for kk in 0..q[0].len() { dot += q[i][kk] * k[j][kk]; }
            scores[i][j] = dot * scale;
        }
    }

    // Softmax over rows: O(n²)
    for i in 0..n {
        let max_val = scores[i].iter().cloned().fold(f32::NEG_INFINITY, f32::max);
        let mut sum = 0.0;
        for j in 0..n {
            scores[i][j] = (scores[i][j] - max_val).exp();
            sum += scores[i][j];
        }
        for j in 0..n { scores[i][j] /= sum; }
    }

    // Attention·V: O(n² · d_v)
    let mut output = vec![vec![0.0f32; d_v]; n];
    for i in 0..n {
        for j in 0..n {
            for kk in 0..d_v {
                output[i][kk] += scores[i][j] * v[j][kk];
            }
        }
    }
    output
}
```

The O(n²) memory and compute in the attention matrix is the bottleneck for long sequences. Flash Attention (Dao et al., 2022) solves this by tiling the computation: process Q·Kᵀ in blocks that fit in GPU SRAM, computing softmax incrementally. This eliminates the O(n²) memory footprint while delivering 2–4× wall-clock speedup through better hardware utilization.

## Softmax

```
yᵢ = exp(xᵢ) / Σⱼ exp(xⱼ)
```

Forward: O(n). The numerical stability trick: subtract max(x) before exp to avoid overflow.

Backward:

```
∂L/∂xᵢ = yᵢ · (∂L/∂yᵢ - Σⱼ yⱼ · ∂L/∂yⱼ)
```

O(n) for the forward, O(n) for the backward. The sum Σ yⱼ·∂L/∂yⱼ is computed once and reused.

## Cross-Entropy Loss

```
L = -log(ŷ_{true_class})
```

Combined with softmax in the last layer, the gradient simplifies beautifully:

```
∂L/∂xᵢ = ŷᵢ - (1 if i == true_class else 0)
```

This is why you always see `CrossEntropyLoss` in PyTorch instead of `LogSoftmax + NLLLoss` — the combined backward pass avoids computing ŷᵢ twice and is numerically more stable.

## Computational Budget by Layer Type

For a typical transformer (BERT-large, 340M parameters, sequence length 512):

| Layer | FLOPs (forward) | % of total |
|-------|----------------|------------|
| Linear (QKV projection) | 6 × 512 × 1024 × 4096 | 38% |
| Attention (Q·Kᵀ + softmax + ·V) | 2 × 512² × 64 × 16 heads | 17% |
| Linear (output projection) | 2 × 512 × 4096 × 1024 | 25% |
| Feed-forward (two linear layers) | 2 × 512 × 4096 × 4096 × 4 | 18% |
| Layer norm + residuals | 2 × 512 × 4096 | 2% |

The linear layers dominate — they're 63% of total FLOPs. Optimizing the matmul kernel (Chapter 11) is still the most important thing you can do for transformer training throughput.

## Fused Kernels

In PyTorch, `LayerNorm + Dropout + Residual` is three separate kernel launches. A **fused kernel** combines them into one GPU kernel, avoiding round-trips to GPU global memory:

```
// Fused residual + layer norm + dropout:
// output = dropout(layer_norm(x + residual))
```

NVIDIA's `apex` library and PyTorch's `torch.nn.functional` provide fused kernels for common patterns. Typical speedup: 20–40% for the affected layers.

Fusing is a micro-optimization, but for models that train for weeks, 20% is days of GPU time saved. It's worth it.
