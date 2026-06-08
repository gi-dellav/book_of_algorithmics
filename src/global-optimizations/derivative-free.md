# Derivative-Free Optimization

When the objective function is a black box — no gradients, possibly noisy, expensive to evaluate — you need derivative-free optimization. This covers hyperparameter tuning, simulation-based design, A/B testing, and any problem where f(x) is the output of an external process.

## Nelder-Mead Simplex Method

The classic derivative-free method maintains a simplex (n+1 points in n dimensions) and iteratively reflects, expands, contracts, or shrinks it toward lower function values:

```rust
fn nelder_mead<F>(f: F, mut simplex: Vec<Vec<f64>>, max_iter: usize,
                   tol: f64) -> Vec<f64>
where F: Fn(&[f64]) -> f64 {
    let n = simplex[0].len();
    let alpha = 1.0f64;   // reflection
    let gamma = 2.0f64;   // expansion
    let rho = 0.5f64;     // contraction
    let sigma = 0.5f64;   // shrink

    for _ in 0..max_iter {
        // Order vertices by function value
        let mut evals: Vec<(f64, Vec<f64>)> = simplex.iter()
            .map(|x| (f(x), x.clone())).collect();
        evals.sort_by(|a, b| a.0.partial_cmp(&b.0).unwrap());

        let x_best = &evals[0].1;
        let x_worst = &evals[n].1;
        let x_second_worst = &evals[n-1].1;

        // Centroid of all points except the worst
        let centroid: Vec<f64> = (0..n).map(|j| {
            (0..n).map(|i| simplex[i][j]).sum::<f64>() / n as f64
        }).collect();

        // Reflect
        let x_reflect: Vec<f64> = centroid.iter().zip(x_worst.iter())
            .map(|(&c, &w)| c + alpha * (c - w)).collect();
        let f_reflect = f(&x_reflect);

        if f_reflect < evals[0].0 {
            // Expand
            let x_expand: Vec<f64> = centroid.iter().zip(x_worst.iter())
                .map(|(&c, &w)| c + gamma * (c - w)).collect();
            let f_expand = f(&x_expand);
            simplex[n] = if f_expand < f_reflect { x_expand } else { x_reflect };
        } else if f_reflect < evals[n-1].0 {
            simplex[n] = x_reflect;
        } else {
            // Contract
            let x_contract: Vec<f64> = centroid.iter().zip(x_worst.iter())
                .map(|(&c, &w)| c + rho * (w - c)).collect();
            let f_contract = f(&x_contract);
            if f_contract < evals[n].0 {
                simplex[n] = x_contract;
            } else {
                // Shrink toward best point
                for i in 1..=n {
                    for j in 0..n {
                        simplex[i][j] = x_best[j] + sigma * (simplex[i][j] - x_best[j]);
                    }
                }
            }
        }

        // Check convergence
        let std: f64 = (simplex.iter().map(|x| f(x)).sum::<f64>() / (n as f64 + 1.0)).sqrt();
        if std < tol { break; }
    }
    simplex[0].clone()
}
```

Nelder-Mead works well for n ≤ 10 and smooth objectives. For n > 20, the simplex tends to collapse. For noisy objectives, it's unreliable. For non-convex problems, it converges to local minima.

## Bayesian Optimization

When each function evaluation is expensive (minutes to hours — e.g., training a neural network, running a physical experiment), Bayesian optimization (BO) builds a probabilistic model (Gaussian process) of f and chooses the next evaluation point to maximally reduce uncertainty about the optimum:

```rust
// Sketch — production BO uses libraries like BoTorch or GPyOpt
fn bayesian_optimization<F>(f: F, bounds: &[(f64, f64)], n_iter: usize) -> Vec<f64>
where F: Fn(&[f64]) -> f64 {
    let n = bounds.len();
    let mut x_obs: Vec<Vec<f64>> = Vec::new();
    let mut y_obs: Vec<f64> = Vec::new();

    // Initial random sampling
    for _ in 0..(2 * n) {
        let x: Vec<f64> = bounds.iter()
            .map(|&(lo, hi)| lo + rand::random::<f64>() * (hi - lo))
            .collect();
        let y = f(&x);
        x_obs.push(x);
        y_obs.push(y);
    }

    let mut gp = GaussianProcess::fit(&x_obs, &y_obs);

    for _ in 0..n_iter {
        // Find x that maximizes the acquisition function (Expected Improvement)
        let x_next = maximize_acquisition(&gp, bounds, &y_obs);
        let y_next = f(&x_next);
        x_obs.push(x_next);
        y_obs.push(y_next);
        gp.update(&x_obs, &y_obs);
    }

    // Return the best observed point
    let best_idx = y_obs.iter().enumerate()
        .min_by(|(_, a), (_, b)| a.partial_cmp(b).unwrap())
        .map(|(i, _)| i).unwrap();
    x_obs[best_idx].clone()
}
```

The acquisition function (Expected Improvement, Upper Confidence Bound, or Probability of Improvement) balances exploration (high uncertainty) and exploitation (low mean). BO typically finds the optimum in 50–200 evaluations — a 100× reduction over random search for n ≤ 20.

## CMA-ES (Covariance Matrix Adaptation Evolution Strategy)

For non-convex, high-dimensional (n = 10–1000) black-box optimization, CMA-ES is the state of the art. It maintains a multivariate normal distribution over the search space and adapts its mean and covariance based on the best-performing samples:

```rust
fn cma_es<F>(f: F, dim: usize, max_iter: usize,
              pop_size: usize) -> Vec<f64>
where F: Fn(&[f64]) -> f64 {
    let mut mean = vec![0.0f64; dim]; // initial guess
    let mut sigma = 0.5f64;           // step size
    let mut c = /* identity covariance */;
    let mu = pop_size / 2;            // number of parents

    for _ in 0..max_iter {
        // Sample population from N(mean, sigma² C)
        let mut population: Vec<(f64, Vec<f64>)> = (0..pop_size)
            .map(|_| {
                let x = sample_multivariate_normal(&mean, sigma, &c);
                (f(&x), x)
            }).collect();

        // Select top μ individuals
        population.sort_by(|a, b| a.0.partial_cmp(&b.0).unwrap());
        let parents: Vec<&Vec<f64>> = population[..mu]
            .iter().map(|(_, x)| x).collect();

        // Update mean (weighted average of parents)
        for i in 0..dim {
            mean[i] = parents.iter().map(|&x| x[i]).sum::<f64>() / mu as f64;
        }

        // Update covariance matrix (rank-μ update)
        update_covariance(&mut c, &parents, &mean, sigma);

        // Update step size (cumulative step-size adaptation)
        sigma = update_step_size(sigma, /* ... */);
    }
    mean
}
```

CMA-ES is the default in Nevergrad, Optuna, and other hyperparameter optimization frameworks. It handles ill-conditioned, non-separable, and multi-modal objectives robustly. For n = 100 and a budget of 10⁴ evaluations, CMA-ES typically finds the global optimum or a very good local optimum.

## When Derivative-Free Methods Win

| Scenario | Method | Budget (typical) |
|----------|--------|-----------------|
| n ≤ 10, smooth, cheap f | Nelder-Mead | 100–1000 eval |
| n ≤ 20, expensive f (minutes) | Bayesian optimization | 50–200 eval |
| n ≤ 100, moderate f (seconds) | CMA-ES | 1000–10000 eval |
| n ≤ 1000, cheap f (milliseconds) | CMA-ES + restarts | 10⁴–10⁵ eval |
| Discrete/categorical params | Random search + local search | 1000–10⁴ eval |
| Multi-objective | NSGA-II, MOEA/D | 10³–10⁵ eval |

A common anti-pattern: using Bayesian optimization for hyperparameters that could be optimized with gradient descent. BO shines when each evaluation costs > 1 minute and n ≤ 20. For cheap evaluations, CMA-ES or random search + local refinement is more effective.

Also: always start with a random search baseline. For many practical problems, random search with 2× the budget of a sophisticated method finds a comparable solution — especially when the objective has low effective dimensionality (most parameters don't matter).
