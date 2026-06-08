# Constrained Optimization

Most real-world optimization problems have constraints: non-negative weights, budget limits, probability simplex, structural bounds. Constrained optimization algorithms enforce these without sacrificing convergence speed. The key methods — Lagrange multipliers, interior-point, and projected gradient — reduce constrained problems to sequences of unconstrained ones.

## Lagrange Multipliers and KKT Conditions

For a problem with equality constraints:

```
minimize f(x)  subject to  h(x) = 0
```

The Lagrangian: L(x, λ) = f(x) + λᵀ h(x). At the optimum, ∇L = 0:

```
∇f(x) + J_h(x)ᵀ λ = 0
h(x) = 0
```

Where J_h is the Jacobian of h. These are the **KKT conditions** (Karush-Kuhn-Tucker). For inequality constraints g(x) ≤ 0, we add complementary slackness:

```
λ_i ≥ 0,  λ_i · g_i(x) = 0
```

## Projected Gradient Descent

For simple constraints (box bounds, simplex), project the gradient step back onto the feasible set:

```rust
fn projected_gradient_descent<F, G, P>(f: F, grad: G, project: P,
                                        mut x: Vec<f64>,
                                        learning_rate: f64,
                                        max_iter: usize) -> Vec<f64>
where F: Fn(&[f64]) -> f64,
      G: Fn(&[f64]) -> Vec<f64>,
      P: Fn(&[f64]) -> Vec<f64> {
    for _ in 0..max_iter {
        let g = grad(&x);
        for i in 0..x.len() { x[i] -= learning_rate * g[i]; }
        x = project(&x);
    }
    x
}

fn project_box(x: &[f64], lower: &[f64], upper: &[f64]) -> Vec<f64> {
    x.iter().zip(lower.iter()).zip(upper.iter())
        .map(|((&xi, &lo), &hi)| xi.max(lo).min(hi))
        .collect()
}

fn project_simplex(x: &[f64]) -> Vec<f64> {
    // Sort x descending, find threshold θ s.t. Σ max(x_i - θ, 0) = 1
    let mut sorted: Vec<f64> = x.to_vec();
    sorted.sort_by(|a, b| b.partial_cmp(a).unwrap());

    let mut cum_sum = 0.0f64;
    let mut theta = 0.0f64;
    for (i, &s) in sorted.iter().enumerate() {
        cum_sum += s;
        let t = (cum_sum - 1.0) / (i as f64 + 1.0);
        if s > t { theta = t; }
    }

    x.iter().map(|&xi| (xi - theta).max(0.0)).collect()
}
```

Projected gradient descent converges at O(1/k) for convex problems — same rate as unconstrained GD. The projection step must be cheap (O(n) for box/simplex, O(n log n) for ℓ₁ ball). When the projection is expensive (e.g., semidefinite cone), interior-point methods are preferred.

## Interior-Point Methods

For problems with inequality constraints g(x) ≥ 0 (reformulate g(x) ≤ 0 if needed), the **logarithmic barrier** method adds a penalty that blows up as x approaches the boundary:

```
minimize f(x) - μ Σ log(g_i(x))
```

As μ → 0, the barrier solution approaches the true constrained optimum. The **primal-dual interior-point method** solves this by tracking the central path:

```rust
fn interior_point<F, G, H>(f: F, grad: G, constraints: &[H],
                            mut x: Vec<f64>, max_iter: usize) -> Vec<f64>
where F: Fn(&[f64]) -> f64,
      G: Fn(&[f64]) -> Vec<f64>,
      H: Fn(&[f64]) -> f64 {
    let n = x.len();
    let m = constraints.len();
    let mut mu = 1.0f64;
    let sigma = 0.1f64; // barrier reduction factor

    for _ in 0..max_iter {
        // Solve KKT system for the barrier subproblem
        let g = grad(&x);
        let mut h = vec![0.0f64; n]; // gradient of barrier term
        let mut hess_diag = vec![0.0f64; m];

        for j in 0..m {
            let gj = constraints[j](&x);
            let gj_inv = 1.0 / gj;
            h.iter_mut().for_each(|hi| *hi -= mu * gj_inv); // simplified
            hess_diag[j] = mu * gj_inv * gj_inv;
        }

        // Newton step for the barrier subproblem
        // (In practice, solve the augmented KKT system directly)
        // ...

        mu *= sigma; // drive μ → 0
    }
    x
}
```

Interior-point methods converge in O(log(1/ε) · √n) iterations for linear programming — polynomial and practically fast. Commercial solvers (Gurobi, Mosek, CPLEX) use highly tuned primal-dual interior-point methods. For open-source use, `clarabel` (Rust) and `cvxpy` (Python) provide interior-point solvers for convex problems.

## SQP (Sequential Quadratic Programming)

For non-convex constrained problems, SQP solves a sequence of quadratic subproblems that locally approximate the objective and linearize the constraints:

```
At iteration k:
  minimize  ½ dᵀ Hₖ d + ∇f(xₖ)ᵀ d
  subject to  ∇h(xₖ)ᵀ d + h(xₖ) = 0
              ∇g(xₖ)ᵀ d + g(xₖ) ≤ 0
```

Hₖ is a (quasi-Newton) approximation of the Hessian of the Lagrangian. The QP subproblem is solved with an active-set or interior-point QP solver. SQP converges quadratically near the optimum if Hₖ is exact.

## Penalty and Augmented Lagrangian Methods

When constraints are expensive to enforce exactly, add a penalty term to the objective:

```
minimize f(x) + (ρ/2) Σ max(0, g_i(x))²
```

As ρ → ∞, the penalty solution approaches feasibility. The **augmented Lagrangian** adds a Lagrange multiplier estimate to avoid needing ρ → ∞:

```
L_A(x, λ; ρ) = f(x) + λᵀ g(x) + (ρ/2) ‖g(x)‖²
```

The augmented Lagrangian method alternates between:
1. Approximately minimize L_A(x, λ; ρ) for fixed λ, ρ.
2. Update λ ← λ + ρ g(x).
3. Possibly increase ρ if feasibility hasn't improved.

This reduces a constrained problem to a sequence of unconstrained problems, each solved with L-BFGS or gradient descent. It's the workhorse for large-scale constrained optimization in machine learning (e.g., training with fairness constraints).

## Which Method for Which Problem?

| Problem type | Method | Solver |
|-------------|--------|--------|
| Box-constrained, smooth | Projected L-BFGS | L-BFGS-B |
| Linear programming | Primal-dual interior-point | Gurobi, HiGHS |
| Quadratic programming | Active-set or interior-point | OSQP, qpOASES |
| Convex, general constraints | Interior-point | CVXOPT, Clarabel |
| Non-convex, equality constraints | SQP | SNOPT, IPOPT |
| Non-convex, inequality + equality | Augmented Lagrangian | LANCELOT, Algencan |
| Large-scale, simple constraints | Projected gradient / Adam | PyTorch, custom |

For 90% of practical problems, L-BFGS-B (box-constrained L-BFGS) or OSQP (QP solver) suffices. For the remaining 10%, reach for IPOPT (interior-point for general non-convex NLPs) or an augmented Lagrangian solver.
