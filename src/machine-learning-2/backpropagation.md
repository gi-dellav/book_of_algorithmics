# Backpropagation Explained Simply

Backpropagation is the chain rule. Nothing more. If you can compute derivatives of scalar functions, you already understand backpropagation — you just need to see how it organizes the computation.

This article builds backpropagation from scratch, starting with a single number and ending with the algorithm that trains GPT-4.

## The Chain Rule in One Minute

If y = f(x) and z = g(y), then:

```
dz/dx = dz/dy · dy/dx
```

That's it. Backpropagation is just this rule applied to every operation in a computation graph, from the output backward.

## A Worked Example: Two-Layer Network

Consider a tiny network:

```
h = W₁x + b₁        (linear layer 1)
a = max(0, h)        (ReLU activation)
ŷ = W₂a + b₂        (linear layer 2)
L = (ŷ - y)²        (squared error loss)
```

Given x (input), y (target), W₁, b₁, W₂, b₂ (parameters), we want ∂L/∂W₁, ∂L/∂b₁, ∂L/∂W₂, ∂L/∂b₂.

### Forward Pass (Compute Everything)

```rust
// Given: x (d_in,), y (d_out,), W1 (d_hidden, d_in), b1 (d_hidden,),
//        W2 (d_out, d_hidden), b2 (d_out,)

// Layer 1
let h = matvec_mul(w1, x);  // h[i] = Σⱼ W1[i][j] * x[j]
let a = h.clone();
for i in 0..d_hidden { a[i] = a[i].max(0.0); }  // ReLU

// Layer 2
let y_hat = matvec_mul(w2, a);  // y_hat[i] = Σⱼ W2[i][j] * a[j]
let loss: f64 = y_hat.iter().zip(y.iter())
    .map(|(&yh, &yt)| (yh - yt) * (yh - yt))
    .sum();
```

### Backward Pass (Gradient Propagation)

We compute gradients by working backward from the loss. At each step, we apply the chain rule.

**Step 1: Gradient of the loss with respect to ŷ**

```
L = Σ (ŷᵢ - yᵢ)²
∂L/∂ŷᵢ = 2(ŷᵢ - yᵢ)
```

```rust
let mut d_y_hat = vec![0.0f64; d_out];
for i in 0..d_out {
    d_y_hat[i] = 2.0 * (y_hat[i] - y[i]);
}
```

**Step 2: Gradient through the second linear layer**

```
ŷ = W₂a + b₂

∂ŷᵢ/∂W₂[i][j] = aⱼ   →   ∂L/∂W₂[i][j] = ∂L/∂ŷᵢ · aⱼ
∂ŷᵢ/∂b₂[i] = 1        →   ∂L/∂b₂[i] = ∂L/∂ŷᵢ
∂ŷᵢ/∂aⱼ = W₂[i][j]    →   ∂L/∂aⱼ = Σᵢ ∂L/∂ŷᵢ · W₂[i][j]
```

```rust
// Gradient for W2: outer product of d_y_hat and a
let mut d_w2 = vec![vec![0.0f64; d_hidden]; d_out];
for i in 0..d_out {
    for j in 0..d_hidden {
        d_w2[i][j] = d_y_hat[i] * a[j];
    }
}

// Gradient for b2: same as d_y_hat
let d_b2 = d_y_hat.clone();

// Gradient for a: W2ᵀ · d_y_hat
let mut d_a = vec![0.0f64; d_hidden];
for j in 0..d_hidden {
    for i in 0..d_out {
        d_a[j] += w2[i][j] * d_y_hat[i];  // note: W2[i][j], not transpose
    }
}
```

**Step 3: Gradient through ReLU**

```
a = max(0, h)
∂aⱼ/∂hⱼ = 1 if hⱼ > 0, else 0
∂L/∂hⱼ = ∂L/∂aⱼ · (1 if hⱼ > 0 else 0)
```

```rust
let mut d_h = vec![0.0f64; d_hidden];
for j in 0..d_hidden {
    d_h[j] = if h[j] > 0.0 { d_a[j] } else { 0.0 };
}
```

**Step 4: Gradient through the first linear layer**

```
h = W₁x + b₁
Same pattern as the second layer.
```

```rust
// Gradient for W1: outer product of d_h and x
let mut d_w1 = vec![vec![0.0f64; d_in]; d_hidden];
for i in 0..d_hidden {
    for j in 0..d_in {
        d_w1[i][j] = d_h[i] * x[j];
    }
}

// Gradient for b1
let d_b1 = d_h.clone();
```

Done. We have ∂L/∂W₁, ∂L/∂b₁, ∂L/∂W₂, ∂L/∂b₂. The backward pass computed exactly the same number of operations as the forward pass (each forward multiply-add has a corresponding backward multiply-add).

## The Pattern: Local Gradients

Every operation in a neural network can be written as `y = f(x₁, x₂, ...)`. To backpropagate through it, we need:

1. The **forward values** (x₁, x₂, ..., y) — stored during the forward pass.
2. The **incoming gradient** ∂L/∂y — passed from the downstream operation.
3. The **local gradients** ∂y/∂x₁, ∂y/∂x₂, ... — computed from the forward values.
4. Output: ∂L/∂xᵢ = ∂L/∂y · ∂y/∂xᵢ (chain rule).

This is why we store intermediate values (`h`, `a`, `y_hat`) during the forward pass — we need them to compute the local gradients during the backward pass. The memory cost of storing all activations is the main memory bottleneck in training (often 3–5× the model parameters).

## Common Operations and Their Local Gradients

| Operation | Forward | Backward (given dL/dy) |
|-----------|---------|------------------------|
| y = x₁ + x₂ | y = x₁ + x₂ | dL/dx₁ = dL/dy, dL/dx₂ = dL/dy |
| y = x₁ · x₂ | y = x₁ · x₂ | dL/dx₁ = dL/dy · x₂, dL/dx₂ = dL/dy · x₁ |
| y = x₁ / x₂ | y = x₁ / x₂ | dL/dx₁ = dL/dy / x₂, dL/dx₂ = -dL/dy · x₁ / x₂² |
| y = max(0, x) | y = max(0, x) | dL/dx = dL/dy if x > 0 else 0 |
| y = exp(x) | y = exp(x) | dL/dx = dL/dy · y |
| y = ln(x) | y = ln(x) | dL/dx = dL/dy / x |
| y = matmul(A, B) | y = A·B | dL/dA = dL/dy · Bᵀ, dL/dB = Aᵀ · dL/dy |
| y = softmax(x) | yᵢ = exp(xᵢ)/Σexp(xⱼ) | dL/dx = y ⊙ (dL/dy - Σ yⱼ·dL/dyⱼ) |

Notice: for elementwise operations (add, multiply, ReLU), the backward pass is also elementwise and cheap. For matrix operations (matmul), the backward pass involves another matrix multiplication — same cost as the forward pass.

## Why Backprop Is O(Forward Cost)

For every operation in the forward pass:
- The backward pass computes the same number of floating-point operations.
- One multiply-add in forward → one multiply-add in backward (for linear ops).

Therefore, the total cost of forward + backward is ~2× the forward cost. Computing the gradient of a neural network takes about as long as evaluating it twice. This is the **cheap gradient principle**: automatic differentiation does not asymptotically increase the computational cost.

Memory, however, is a different story. The forward pass must store every intermediate activation for use in the backward pass. For a transformer with 175B parameters (GPT-3), storing activations for a sequence of 2048 tokens requires ~3 TB of memory — far exceeding GPU HBM (80 GB on A100). The solution: **activation checkpointing** (trade compute for memory by recomputing some activations during the backward pass).

## From Backprop to Training

Putting it together:

```rust
fn train_step(w1: &mut [Vec<f64>], b1: &mut [f64],
              w2: &mut [Vec<f64>], b2: &mut [f64],
              x: &[f64], y: &[f64], learning_rate: f64) -> f64 {
    // Forward pass
    let (loss, h, a, y_hat) = forward(w1, b1, w2, b2, x, y);

    // Backward pass
    let (d_w1, d_b1, d_w2, d_b2) = backward(w1, w2, &h, &a, &y_hat, y, x);

    // Update parameters (SGD step)
    for i in 0..w1.len() {
        for j in 0..w1[0].len() {
            w1[i][j] -= learning_rate * d_w1[i][j];
        }
        b1[i] -= learning_rate * d_b1[i];
    }
    for i in 0..w2.len() {
        for j in 0..w2[0].len() {
            w2[i][j] -= learning_rate * d_w2[i][j];
        }
        b2[i] -= learning_rate * d_b2[i];
    }

    loss
}
```

This is the entire training algorithm. Everything else — Adam, batch normalization, residual connections, attention — is just more operations in the forward pass with their corresponding backward rules. The chain rule handles them all uniformly.

## The Big Picture

Backpropagation is beautiful because it's so simple. The chain rule from calculus 101, applied systematically from output to input, computes all the gradients you need. The only "trick" is storing intermediate values on the forward pass so you can use them on the backward pass. Everything else is engineering.
