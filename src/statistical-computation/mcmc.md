# MCMC and Gibbs Sampling

Markov Chain Monte Carlo (MCMC) is the engine of Bayesian statistics. When you can't compute a posterior distribution analytically, MCMC constructs a Markov chain whose stationary distribution is the posterior. Run the chain long enough, and your samples are draws from the posterior. The computational challenge: the chain may need millions of steps, and each step requires evaluating the likelihood — which is O(n) in the data size.

## Metropolis-Hastings

The simplest MCMC algorithm. Given current state θ, propose a new state θ' from a proposal distribution q(θ'|θ). Accept with probability:

```
α = min(1, p(θ') · q(θ|θ') / (p(θ) · q(θ'|θ)))
```

```rust
fn metropolis_hastings<F, G>(log_posterior: F, proposal: G,
                               theta_init: Vec<f64>, n_iter: usize,
                               rng: &mut impl Rng) -> Vec<Vec<f64>>
where F: Fn(&[f64]) -> f64,
      G: Fn(&[f64], &mut impl Rng) -> Vec<f64> {
    let d = theta_init.len();
    let mut chain = Vec::with_capacity(n_iter);
    let mut theta = theta_init.clone();
    let mut log_p_curr = log_posterior(&theta);

    for _ in 0..n_iter {
        let theta_prop = proposal(&theta, rng);
        let log_p_prop = log_posterior(&theta_prop);

        let log_alpha = (log_p_prop - log_p_curr).min(0.0); // symmetric proposal
        if rng.gen::<f64>().ln() < log_alpha {
            theta = theta_prop;
            log_p_curr = log_p_prop;
        }
        chain.push(theta.clone());
    }
    chain
}
```

The bottleneck is `log_posterior`, which for a model with n data points is O(n) per iteration. For n = 10⁶ and 10⁴ iterations: 10¹⁰ likelihood evaluations — minutes to hours.

## Optimization 1: Random-Scan Gibbs

When the posterior factorizes nicely, Gibbs sampling updates one parameter at a time, conditioning on the others. This avoids the accept/reject step (always accept) and can be much faster:

```rust
fn gibbs_sampling(data: &[f64], n_iter: usize) -> Vec<(f64, f64)> {
    let n = data.len();
    let mut mu = 0.0f64;   // current estimate of mean
    let mut tau = 1.0f64;  // current precision (1/variance)
    let mut chain = Vec::with_capacity(n_iter);

    let sum_x: f64 = data.iter().sum();
    let sum_x2: f64 = data.iter().map(|&x| x * x).sum();

    for _ in 0..n_iter {
        // Sample mu | tau, data
        let post_var = 1.0 / (tau * n as f64 + 1.0);
        let post_mean = post_var * (tau * sum_x);
        mu = post_mean + post_var.sqrt() * thread_rng().sample::<f64, _>(
            rand_distr::StandardNormal
        );

        // Sample tau | mu, data
        let shape = n as f64 / 2.0 + 1.0;
        let rate = 0.5 * (sum_x2 - 2.0 * mu * sum_x + n as f64 * mu * mu) + 1.0;
        tau = rand_distr::Gamma::new(shape, 1.0 / rate).unwrap()
            .sample(&mut thread_rng());

        chain.push((mu, tau));
    }
    chain
}
```

Gibbs sampling uses sufficient statistics (`sum_x`, `sum_x2`) computed once — each iteration is O(1), not O(n). This is the key to fast MCMC: precompute what you can and update incrementally.

## Gradient-Based MCMC: HMC and NUTS

Hamiltonian Monte Carlo uses gradient information to propose distant states with high acceptance probability. Instead of a random walk (which explores the posterior at O(1/√k) efficiency), HMC simulates Hamiltonian dynamics:

```rust
fn hmc_step<F>(log_posterior: F, grad_log_posterior: G,
               theta: &mut Vec<f64>, step_size: f64, n_leapfrog: usize,
               rng: &mut impl Rng)
where F: Fn(&[f64]) -> f64,
      G: Fn(&[f64]) -> Vec<f64> {
    let d = theta.len();

    // Sample momentum from standard normal
    let mut r: Vec<f64> = (0..d)
        .map(|_| rng.sample(rand_distr::StandardNormal))
        .collect();
    let r_init = r.clone();

    // Leapfrog integration
    let mut theta_cur = theta.clone();
    let grad = grad_log_posterior(&theta_cur);
    for i in 0..d { r[i] += 0.5 * step_size * grad[i]; }

    for _ in 0..n_leapfrog {
        for i in 0..d { theta_cur[i] += step_size * r[i]; }
        let grad = grad_log_posterior(&theta_cur);
        for i in 0..d { r[i] += step_size * grad[i]; }
    }

    let grad = grad_log_posterior(&theta_cur);
    for i in 0..d { r[i] += 0.5 * step_size * grad[i]; }
    // Negate momentum for reversibility
    for i in 0..d { r[i] = -r[i]; }

    // Metropolis accept/reject
    let log_p_cur = log_posterior(theta);
    let log_p_prop = log_posterior(&theta_cur);
    let kinetic_cur = r_init.iter().map(|&ri| ri * ri).sum::<f64>() / 2.0;
    let kinetic_prop = r.iter().map(|&ri| ri * ri).sum::<f64>() / 2.0;

    let log_alpha = (log_p_prop - kinetic_prop) - (log_p_cur - kinetic_cur);
    if rng.gen::<f64>().ln() < log_alpha.min(0.0) {
        *theta = theta_cur;
    }
}
```

The gradient `grad_log_posterior` is O(n) per leapfrog step, and we need L ≈ 10–20 leapfrog steps per HMC step. That's 10–20× the cost of Metropolis-Hastings. But HMC's acceptance rate is ~90% vs. ~23% for optimally-tuned Metropolis-Hastings, and the proposals are much further from the current state — producing less autocorrelated samples. The effective sample size per gradient evaluation is often 10× higher.

## NUTS (No-U-Turn Sampler)

HMC requires tuning the number of leapfrog steps L. NUTS (Hoffman & Gelman, 2014) adaptively chooses L by simulating the trajectory forward and backward until it makes a "U-turn" (the momentum would send it back toward the start):

```
Build a binary tree of states by repeatedly doubling the trajectory length.
Stop when the trajectory starts to loop back (the farthest state is no longer moving away).
Sample from the tree using detailed balance.
```

NUTS is the default in Stan, PyMC, and Turing.jl. It eliminates the need to tune L and step_size (which is adapted separately using dual averaging).

## Optimization 2: Subsampling the Likelihood

For large n, computing the full log-likelihood at every step is infeasible. **Stochastic Gradient Langevin Dynamics** (SGLD) uses a mini-batch:

```
θ_{t+1} = θ_t + ε_t/2 · (∇ log p(θ_t) + n/|B| · Σ_{i∈B} ∇ log p(y_i|θ_t)) + η_t
```

Where η_t ~ N(0, ε_t). The mini-batch gradient approximates the full gradient, and the added noise ensures the chain converges to the correct posterior (as ε_t → 0). This reduces the per-iteration cost from O(n) to O(|B|) — for |B| = 100 and n = 10⁶, that's a 10,000× speedup per step.

```rust
fn sgld_step(data: &[(f64, f64)], theta: &mut Vec<f64>,
             step_size: f64, batch_size: usize, rng: &mut impl Rng) {
    let n = data.len();
    let batch: Vec<_> = (0..batch_size)
        .map(|_| data[rng.gen_range(0..n)])
        .collect();

    let grad = stochastic_gradient(&batch, theta, n);
    for i in 0..theta.len() {
        theta[i] += 0.5 * step_size * grad[i];
        theta[i] += step_size.sqrt() * rng.sample(rand_distr::StandardNormal);
    }
}
```

SGLD is asymptotically exact as step_size → 0 and n_iter → ∞. In practice, use a decreasing step size (ε_t = a · (b + t)^{-γ}) and compute diagnostics (potential scale reduction factor R̂) to assess convergence.

## When to Use What

| Problem | Method | Per-iteration cost |
|---------|--------|-------------------|
| Small n, any model | NUTS/HMC | O(n) |
| Large n, differentiable | SGLD | O(batch_size) |
| Conditionally conjugate | Gibbs | O(1) after precomputation |
| Intractable likelihood | Approximate Bayesian Computation (ABC) | O(n · n_sim) |
| Discrete parameters | Metropolis-Hastings | O(n) |
| Mixed discrete/continuous | Reversible-jump MCMC | O(n) |

## Convergence Diagnostics

Don't trust a chain without diagnostics:

- **R̂ (Gelman-Rubin)**: Run multiple chains from overdispersed starting points. R̂ compares within-chain and between-chain variance. R̂ < 1.01 indicates convergence.
- **Effective sample size**: n_eff = n / (1 + 2 Σ ρ_k) where ρ_k is the autocorrelation at lag k. ESS > 400 is adequate for most inference.
- **Trace plots**: Visual inspection for trends, stuck chains, and multimodality.

```rust
fn gelman_rubin(chains: &[Vec<f64>]) -> f64 {
    let m = chains.len() as f64;
    let n = chains[0].len() as f64;

    let chain_means: Vec<f64> = chains.iter()
        .map(|c| c.iter().sum::<f64>() / n).collect();
    let grand_mean = chain_means.iter().sum::<f64>() / m;

    let b = n / (m - 1.0) * chain_means.iter()
        .map(|&cm| (cm - grand_mean) * (cm - grand_mean)).sum::<f64>();

    let w = chains.iter().map(|c| {
        let mean = c.iter().sum::<f64>() / n;
        c.iter().map(|&x| (x - mean) * (x - mean)).sum::<f64>() / (n - 1.0)
    }).sum::<f64>() / m;

    let var_hat = (n - 1.0) / n * w + b / n;
    (var_hat / w).sqrt()
}
```

R̂ > 1.1 means the chains haven't mixed — run longer or reparameterize.
