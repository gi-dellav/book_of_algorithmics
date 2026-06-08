# Automatic Differentiation

Backpropagation is the algorithm. Automatic differentiation (autodiff) is the software that implements it for arbitrary code. When you write `loss.backward()` in PyTorch, you invoke an autodiff engine that traces the computation graph and computes gradients without you writing a single derivative.

## The Two Flavors of Autodiff

**Forward-mode**: compute derivatives alongside the forward computation. Given x, propagate both value and derivative: `(y, ẏ) = f(x, ẋ)`. Efficient when output dimension ≫ input dimension (computing Jacobian-vector products). Used for: ODE sensitivity analysis, Hessian-vector products.

**Reverse-mode**: record the computation graph during forward, then traverse it backward computing gradients. Efficient when output dimension ≪ input dimension (scalar loss → millions of parameters). Used for: neural network training.

This article focuses on reverse-mode autodiff, the workhorse of deep learning.

## Wengert Lists (Tape-Based Autodiff)

The simplest implementation: during the forward pass, record every operation and its inputs into a **tape** (Wengert list). During backward, replay the tape in reverse:

```rust
enum Op {
    Add(usize, usize),       // out = a + b      (indices into value array)
    Mul(usize, usize),       // out = a * b
    ReLU(usize),             // out = max(0, a)
    MatMul(usize, usize),    // out = a @ b
}

struct Tape {
    ops: Vec<Op>,
    values: Vec<f64>,         // forward values (size grows with each op)
}

impl Tape {
    fn add(&mut self, a: usize, b: usize) -> usize {
        let result = self.values[a] + self.values[b];
        let idx = self.values.len();
        self.values.push(result);
        self.ops.push(Op::Add(a, b));
        idx
    }

    fn backward(&self) -> Vec<f64> {
        let n_vals = self.values.len();
        let mut grad = vec![0.0f64; n_vals];
        grad[n_vals - 1] = 1.0; // gradient of output w.r.t itself

        for op in self.ops.iter().rev() {
            match *op {
                Op::Add(a, b) => {
                    // y = a + b → ∂L/∂a += ∂L/∂y, ∂L/∂b += ∂L/∂y
                    let idx = /* result index */;
                    grad[a] += grad[idx];
                    grad[b] += grad[idx];
                }
                Op::Mul(a, b) => {
                    // y = a * b → ∂L/∂a += ∂L/∂y * b, ∂L/∂b += ∂L/∂y * a
                    let idx = /* result index */;
                    grad[a] += grad[idx] * self.values[b];
                    grad[b] += grad[idx] * self.values[a];
                }
                Op::ReLU(a) => {
                    let idx = /* result index */;
                    if self.values[a] > 0.0 {
                        grad[a] += grad[idx];
                    }
                }
                Op::MatMul(a, b) => {
                    // y = a @ b → grad_a += grad_y @ bᵀ, grad_b += aᵀ @ grad_y
                    // (Index-based version requires storing shapes)
                }
            }
        }
        grad
    }
}
```

The tape is simple but stores every intermediate value (memory-hungry) and uses dynamic dispatch for each operation (slow). Production autodiff engines (PyTorch's Autograd, JAX's tracing) are more sophisticated.

## PyTorch's Autograd: DAG of Nodes

PyTorch builds a directed acyclic graph (DAG) where nodes are tensors and edges are operations. Each tensor records its `grad_fn` — the operation that created it:

```python
x = torch.tensor([2.0], requires_grad=True)
y = x * 3        # y.grad_fn = <MulBackward0>
z = y.sin()      # z.grad_fn = <SinBackward0>
z.backward()     # Traverses: SinBackward → MulBackward → leaf x
# x.grad = cos(6) * 3 = 2.88...
```

```rust
// Simplified Rust equivalent
struct Tensor {
    data: Vec<f64>,
    grad: RefCell<Option<Vec<f64>>>,
    grad_fn: Option<Box<dyn BackwardFn>>,
    requires_grad: bool,
}

trait BackwardFn {
    fn apply(&self, grad_output: &[f64]) -> Vec<Vec<f64>>;
    // Return gradients w.r.t each input
}

fn backward(root: &Tensor) {
    let mut grad = root.grad.borrow_mut();
    *grad = Some(vec![1.0; root.data.len()]);

    // Topological sort (reverse order of creation)
    let mut topo = Vec::new();
    build_topo(root, &mut topo, &mut HashSet::new());

    for node in topo.iter().rev() {
        if let Some(ref grad_fn) = node.grad_fn {
            let grad_out = node.grad.borrow().clone().unwrap();
            let grads_in = grad_fn.apply(&grad_out);
            // Accumulate grads_in into the input nodes' .grad fields
            // ...
        }
    }
}
```

The key: each operation knows how to compute its own local gradients. `MulBackward` multiplies by the other input. `SinBackward` multiplies by cos(output). The autograd engine just calls them in the right order.

## Source Transformation (JAX/Zygote Style)

Instead of tracing at runtime, source transformation differentiates the code itself. Given a function `f`, produce a new function `∇f` that computes both value and gradient:

```julia
# Zygote.jl (Julia)
f(x) = sin(x * 3)
f'(2.0)  # Returns cos(6.0) * 3.0 = 2.88...
```

The compiler transforms the AST of `f`, inserting gradient computations alongside the original code. Advantages: no runtime tracing overhead, can differentiate through control flow (if/for) naturally. Disadvantages: harder to implement, can generate large code (the "gradient blowup" problem).

## Checkpointing (Activation Recomputation)

For large models, storing all activations is infeasible. Checkpointing trades compute for memory:

```rust
fn checkpointed_forward<T>(f: impl Fn(T) -> T, x: T) -> (T, impl Fn(T) -> T) {
    // Forward: only save the input, not intermediate activations
    let y = f(x.clone());

    // Return output + a closure that recomputes forward and backward
    let backward_fn = move |grad_y: T| {
        // Re-run forward, this time recording activations
        let (y_recomp, tape) = record_forward(&f, x.clone());
        // Now run backward using the tape
        backward_from_tape(&tape, grad_y)
    };

    (y, Box::new(backward_fn))
}
```

PyTorch's `torch.utils.checkpoint` does exactly this. For a transformer with 100 layers, checkpointing every layer reduces activation memory from O(L × hidden × seq_len) to O(hidden × seq_len) at the cost of ~33% extra compute (each layer's forward is computed twice: once during the forward pass, once during the backward pass).

## The Computational Cost of Autodiff

| Method | Forward cost | Backward cost | Memory |
|--------|-------------|--------------|--------|
| Tape (Wengert list) | O(f) | O(f) | O(f) — stores every intermediate |
| DAG with drop-inactive | O(f) | O(f) | O(active_activations) |
| Checkpointing (every layer) | O(2f) | O(f) | O(1 layer) |
| Source transformation | O(f) | O(2–5× f) | O(f) (compiler-optimized) |

The 2× forward cost of checkpointing is an excellent trade — 33% more compute for ~L× less memory. For L=100, that's 100× memory reduction. This is how models with 175B parameters are trained on GPUs with 80 GB memory.

## Why You Should Understand Autodiff

When `loss.backward()` fails silently (NaN gradients, zero gradients, exploding gradients), understanding the computation graph lets you debug. Key tools:

- `.retain_grad()` — keep gradients for non-leaf tensors.
- `.register_hook(lambda g: print(g.norm()))` — inspect gradients mid-graph.
- `torch.autograd.detect_anomaly()` — find the first operation that produces NaN.

Backprop is simple math. Autodiff is simple engineering. Together, they make training arbitrarily complex networks possible — and debuggable.
