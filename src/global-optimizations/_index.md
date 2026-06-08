# Finding the Best (in High Dimensions)

Optimization is the engine of every machine learning model, every control system, every portfolio allocation, and every engineering design. It's also where naive algorithms fail most spectacularly — gradient descent on a poorly-conditioned problem can take 10⁶ iterations where a quasi-Newton method takes 10.

This chapter bridges the gap from the numeric linear algebra and statistical computation chapters to the machine learning chapters that follow. We cover the optimization algorithms that work at scale: gradient descent, stochastic methods, quasi-Newton, constrained optimization, and derivative-free methods when gradients don't exist.

## What This Chapter Covers

1. **Gradient Descent and Acceleration** — Batch vs. mini-batch vs. stochastic. Momentum, Nesterov acceleration, and Adam. Convergence rates and when each matters.
2. **Quasi-Newton Methods** — BFGS and L-BFGS. Why storing an n×n Hessian approximation is infeasible, and how L-BFGS approximates it with O(n) memory. The gold standard for smooth optimization up to ~10⁶ parameters.
3. **Constrained Optimization** — Lagrange multipliers, KKT conditions, and interior-point methods. When your parameters must satisfy constraints: linear, convex, and non-convex.
4. **Derivative-Free Optimization** — Nelder-Mead, Bayesian optimization, and evolutionary strategies. When the objective is a black box (hyperparameter tuning, simulation-based design).

## Recommended Reading Order

Start with **Gradient Descent** — it's the foundation for everything else. Then quasi-Newton for medium-scale smooth problems, constrained optimization for real-world constraints, and derivative-free for black-box settings.

Cross-reference with Chapter 6 (Arithmetic) for the floating-point implications of gradient accumulation, Chapter 13 (Parallel Computing) for distributed optimization, and the Machine Learning chapters (coming next) for applications.
