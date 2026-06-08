# Numbers from Data

Statistical computation is different from the numeric linear algebra of the previous chapter. Linear algebra solves deterministic equations. Statistics estimates parameters from noisy observations. The algorithms are similar — matrix factorizations, iterative solvers, optimizers — but the emphasis shifts from exact solutions to uncertainty quantification. And the data sizes are often larger: millions of observations × thousands of predictors.

This chapter covers the computational kernels that power modern statistics: distribution sampling, sufficient statistics, regression diagnostics, bootstrap and permutation tests, and Markov Chain Monte Carlo. Each section emphasizes the hardware-aware techniques that turn O(n³) weeks into O(n log n) hours.

## What This Chapter Covers

1. **Random Sampling and Monte Carlo** — Inverse transform sampling, rejection sampling, Ziggurat method for the normal distribution. Generating 10⁹ random variates/second with SIMD.
2. **OLS Regression and Diagnostics** — The QR-based OLS solver. Leverage, Cook's distance, and variance inflation factor — all O(np²) from the QR factorization without re-fitting. The fast leave-one-out formula (PRESS statistic).
3. **Bootstrap and Permutation Tests** — The computational tricks: importance sampling, balanced bootstrap, and parallel resampling. When 1000 bootstrap samples take as long as 10.
4. **MCMC and Gibbs Sampling** — Metropolis-Hastings, Hamiltonian Monte Carlo, and the No-U-Turn Sampler. The computational bottleneck is gradient evaluation — how to make it fast.

## Recommended Reading Order

Read **Random Sampling** first — every other article needs random numbers. Then OLS (the most common regression), Bootstrap (the most common uncertainty quantification), and MCMC (for Bayesian models).

Cross-reference with Chapter 6 (Arithmetic) for floating-point precision in sums, Chapter 8 (External Memory) for out-of-core regression, and Chapter 11 (Linear Algebra) for the QR/Cholesky factorizations used throughout.
