# Logistic Regression Inference

Logistic regression is the workhorse of classification problems — spam detection, medical diagnosis, MNIST digit recognition. Inference (forward pass) is a dot product followed by a sigmoid. This article explores optimizations for batch inference where throughput matters more than latency.

## The Baseline

For a 10-class MNIST classifier with 784 input features and 10 output classes:

```rust
fn logistic_baseline(x: &[f32], w: &[f32], b: &[f32], probs: &mut [f32], n_features: usize, n_classes: usize) {
    for c in 0..n_classes {
        let mut logit = b[c];
        for i in 0..n_features {
            logit += w[c * n_features + i] * x[i];
        }
        probs[c] = logit.exp();
    }
    // Softmax: normalize
    let sum: f32 = probs[..n_classes].iter().sum();
    for c in 0..n_classes {
        probs[c] /= sum;
    }
}
```

For a single image: 784×10 = 7,840 multiply-adds + 10 exps + softmax. ~1 µs on Zen 2. For batch inference (1000 images): ~1 ms.

## Stage 1: Removing the Softmax

For classification, we only need the *argmax* of the probabilities, not the probabilities themselves. The softmax normalization is monotonic — it doesn't change the ordering:

```
argmax(softmax(logits)) = argmax(logits)
```

So we can skip the softmax entirely and just take the argmax of the logits. This eliminates 10 exponential calls and the normalization loop.

```rust
fn classify(x: &[f32], w: &[f32], b: &[f32], n_features: usize, n_classes: usize) -> usize {
    let mut best_class = 0usize;
    let mut best_logit = f32::NEG_INFINITY;
    for c in 0..n_classes {
        let mut logit = b[c];
        for i in 0..n_features {
            logit += w[c * n_features + i] * x[i];
        }
        if logit > best_logit {
            best_logit = logit;
            best_class = c;
        }
    }
    best_class
}
```

Performance: ~1.8× faster (no exp, no softmax normalization, and the argmax loop is cheaper). But we still do the full dot product for each class.

## Stage 2: SIMD Dot Product

The inner loop is a dot product — perfect for SIMD:

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

let mut logit = b[c];
let mut vsum = _mm256_setzero_ps();
let mut i = 0;
while i < n_features {
    let vx = _mm256_loadu_ps(x.as_ptr().add(i));
    let vw = _mm256_loadu_ps(w.as_ptr().add(c * n_features + i));
    vsum = _mm256_fmadd_ps(vx, vw, vsum);
    i += 8;
}
// Horizontal sum of vsum
logit += horizontal_sum_ps(vsum);
```

Performance: ~4× faster than scalar dot product.

## Stage 3: Quantization (Char/Byte Weights)

If we can tolerate slightly lower accuracy, we can store weights as 8-bit signed integers instead of 32-bit floats:

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

let w_quantized: &[i8];  // Pre-quantized: round(W * 127.0f / max_abs_weight)
let scale: f32;          // max_abs_weight / 127.0f (precomputed)

let mut logit = b[c];
let mut vsum_int = _mm256_setzero_si256();
let mut i = 0;
while i < n_features {
    let vw = _mm256_loadu_si256(
        w_quantized.as_ptr().add(c * n_features + i) as *const __m256i
    );
    // Sign-extend weights from int8 to int16, then to int32 (via pmaddubsw + pmaddwd)
    // or use _mm256_maddubs_epi16 for 8-bit × 8-bit → 16-bit accumulation
    // ... 
    i += 32;
}
let logit_float = horizontal_sum_int(vsum_int) as f32 * scale + b[c];
```

The `_mm256_maddubs_epi16` instruction (SSSE3) does 32 8-bit × 8-bit multiplications and accumulates adjacent pairs into 16-bit results — all in one instruction.

Performance: ~2× faster than float SIMD (fewer memory accesses: 4× less data per weight, PLUS wider vectors: 32 elements per load vs. 8 for float).

Accuracy impact: For MNIST, 8-bit weights give ~99.0% accuracy vs. 99.2% for float32. For most applications, the 0.2% accuracy loss is acceptable for the 2× speedup.

## Stage 4: Weight Reordering for Cache

The weight matrix W is stored as `W[class][feature]`. The inner loop reads `W[c * n_features + i]` — sequential for a given class, but touching only 1/n_classes of the matrix per class. For n_features = 784 and n_classes = 10, the full matrix is only 31 KB — it fits in L1. But for larger models (1000 classes, 1000 features), the matrix is 4 MB — fits in L3 but not L1.

Reordering: store weights as `W[feature][class]` (SoA layout). The inner loop reads `W[i * n_classes + c]` — still sequential, but now all classes for a given feature are adjacent. Different access pattern, better for some batching strategies.

Or: block the weight matrix into tiles that fit in L1, and process all classes within a tile before moving to the next.

## Stage 5: Mixed Precision (bfloat16)

bfloat16 (see `arithmetic/ieee-754.md`) halves memory bandwidth while preserving the dynamic range of float32:

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

// Convert bfloat16 to float32: shift left by 16
unsafe fn bf16_to_f32(bf16_lo: __m128i, bf16_hi: __m128i) -> __m256 {
    let mut v = _mm256_insertf128_si256(_mm256_castsi128_si256(bf16_lo), bf16_hi, 1);
    _mm256_castsi256_ps(_mm256_slli_epi32::<16>(v))
}
```

bfloat16 dot product: load 16 BF16 weights (128 bits), convert to 16 float32s (256 bits × 2 registers), FMA with float32 input. Twice as many weights per memory access.

Performance: ~1.5× over int8 quantization (when memory bandwidth is the bottleneck), with slightly better accuracy. The tradeoff depends on hardware support — Zen 4 has native BF16 instructions; Zen 2 requires the conversion dance.

## Performance Summary (1M images, 784×10 model, Zen 2)

| Method | Time | Images/sec | Speedup |
|--------|------|------------|---------|
| Baseline (float, scalar) | 1.0 ms/batch | 1,000 | 1× |
| No softmax | 0.56 ms | 1,800 | 1.8× |
| SIMD dot product | 0.25 ms | 4,000 | 4× |
| Int8 quantization | 0.12 ms | 8,300 | 8.3× |
| BF16 + SIMD | 0.09 ms | 11,000 | 11× |

The optimizations compound: algorithmic (no softmax), vectorization (SIMD), and data compression (int8/BF16). The final version is ~11× faster than the naive implementation while producing identical classification results (argmax of logits is unchanged; quantization introduces negligible accuracy loss).
