# DP Optimizations

Some DP recurrences have hidden structure — monotonicity, convexity, or quadrangle inequality — that lets us reduce O(n³) to O(n²) or O(n log n). These optimizations are algorithmic, not hardware-level, but they produce speedups of 100–1000× that no amount of cache tuning can match.

## Knuth Optimization

For recurrences of the form:

```
dp[i][j] = min_{k in [i, j-1]} (dp[i][k] + dp[k+1][j]) + cost[i][j]
```

If `cost[i][j]` satisfies the **quadrangle inequality**:

```
cost[a][c] + cost[b][d] ≤ cost[a][d] + cost[b][c]  for a ≤ b ≤ c ≤ d
```

And is **monotone** on the lattice of intervals, then the optimal `k` for `dp[i][j]`, denoted `opt[i][j]`, satisfies:

```
opt[i][j-1] ≤ opt[i][j] ≤ opt[i+1][j]
```

This reduces the inner loop from O(n) to amortized O(1), bringing total time from O(n³) to O(n²):

```rust
fn knuth_dp(cost: &[Vec<u32>], n: usize) -> (Vec<Vec<u32>>, Vec<Vec<usize>>) {
    let mut dp = vec![vec![u32::MAX; n]; n];
    let mut opt = vec![vec![0usize; n]; n];

    for i in 0..n {
        dp[i][i] = 0;
        opt[i][i] = i;
    }

    for len in 2..=n {
        for i in 0..=n - len {
            let j = i + len - 1;
            let k_lo = opt[i][j-1];
            let k_hi = opt[i+1][j];
            for k in k_lo..=k_hi {
                let val = dp[i][k].saturating_add(dp[k+1][j])
                    .saturating_add(cost[i][j]);
                if val < dp[i][j] {
                    dp[i][j] = val;
                    opt[i][j] = k;
                }
            }
        }
    }
    (dp, opt)
}
```

The inner `k` loop now ranges over `opt[i+1][j] - opt[i][j-1]` entries, which is O(1) on average. Total operations: O(n²). This is the difference between running in 100 ms (n=5000) and 2 minutes (n=5000, O(n³)).

When does this apply? Optimal binary search trees (with sorted keys), breaking a string into palindromes, optimal polygon triangulation — any problem where the cost of merging two intervals is monotone and satisfies quadrangle inequality.

## Divide-and-Conquer DP

For recurrences of the form:

```
dp[t][i] = min_{j < i} (dp[t-1][j] + cost[j][i])
```

Where `cost[j][i]` satisfies quadrangle inequality (or more weakly, where `opt[t][i]` is monotone in i), we can use divide-and-conquer:

```rust
fn divide_conquer_dp(dp: &mut [Vec<u32>], t: usize,
                     l: usize, r: usize,
                     opt_l: usize, opt_r: usize,
                     cost: &dyn Fn(usize, usize) -> u32) {
    if l > r { return; }
    let mid = (l + r) / 2;

    // Find best split for mid
    let mut best = u32::MAX;
    let mut best_k = opt_l;
    for k in opt_l..=opt_r.min(mid - 1) {
        let val = dp[t-1][k].saturating_add(cost(k, mid));
        if val < best {
            best = val;
            best_k = k;
        }
    }
    dp[t][mid] = best;

    // Recurse with monotonic bounds
    divide_conquer_dp(dp, t, l, mid - 1, opt_l, best_k, cost);
    divide_conquer_dp(dp, t, mid + 1, r, best_k, opt_r, cost);
}
```

At each level of recursion, we search `opt_r - opt_l + 1` candidates spread across `r - l + 1` positions. Total: O(n log n) per row, vs. O(n²) for the naive nested loop. For n = 100,000 and T = 50 rows, this reduces runtime from 10 minutes to 0.5 seconds.

## Convex Hull Trick

For recurrences of the form:

```
dp[i] = min_{j < i} (dp[j] + m[j] * x[i] + b[j])
```

Where `m[j]` are monotone (increasing or decreasing) and queries `x[i]` are monotone, we maintain the **lower envelope** of lines `y = m[j] * x + b[j]`:

```rust
struct Line {
    m: f64,
    b: f64,
}

fn intersect(l1: &Line, l2: &Line) -> f64 {
    (l2.b - l1.b) / (l1.m - l2.m)
}

fn convex_hull_dp(m: &[f64], b: &[f64], x: &[f64]) -> Vec<f64> {
    let n = x.len();
    let mut dp = vec![0.0f64; n];
    let mut hull: Vec<Line> = Vec::new();
    let mut ptr = 0usize; // pointer to best line

    for i in 0..n {
        // Add line for dp[i] (if it forms part of the lower envelope)
        let new_line = Line { m: m[i], b: b[i] + if i > 0 { dp[i-1] } else { 0.0 } };

        while hull.len() >= 2 {
            let l1 = &hull[hull.len() - 2];
            let l2 = &hull[hull.len() - 1];
            if intersect(l1, l2) >= intersect(l2, &new_line) {
                hull.pop();
            } else {
                break;
            }
        }
        hull.push(new_line);

        // Query: find best line for x[i] (monotone x)
        while ptr + 1 < hull.len()
              && hull[ptr].m * x[i] + hull[ptr].b >= hull[ptr+1].m * x[i] + hull[ptr+1].b {
            ptr += 1;
        }
        dp[i] = hull[ptr].m * x[i] + hull[ptr].b;
    }
    dp
}
```

The convex hull trick reduces O(n²) to O(n) for DP with linear transitions. Applications: 1D cutting stock, optimal batch processing, piecewise-linear optimization.

## Li Chao Segment Tree

When queries `x[i]` are NOT monotone (or lines are added in arbitrary order), use a Li Chao segment tree: O(log C) per query and insertion, where C is the coordinate range:

```rust
struct LiChaoNode {
    line: Option<Line>,
    left: Option<Box<LiChaoNode>>,
    right: Option<Box<LiChaoNode>>,
}

impl LiChaoNode {
    fn insert(&mut self, l: i64, r: i64, mut line: Line) {
        if self.line.is_none() {
            self.line = Some(line);
            return;
        }
        let mid = (l + r) / 2;
        let cur = self.line.as_mut().unwrap();

        let left_better = line.eval(l) < cur.eval(l);
        let mid_better = line.eval(mid) < cur.eval(mid);

        if mid_better { std::mem::swap(cur, &mut line); }
        if r - l <= 1 { return; }

        if left_better != mid_better {
            self.left.get_or_insert_with(|| Box::new(LiChaoNode::new()))
                .insert(l, mid, line);
        } else {
            self.right.get_or_insert_with(|| Box::new(LiChaoNode::new()))
                .insert(mid, r, line);
        }
    }

    fn query(&self, l: i64, r: i64, x: i64) -> Option<f64> {
        let mut ans = self.line.as_ref().map(|ln| ln.eval(x));
        let mid = (l + r) / 2;
        if x < mid {
            if let Some(ref left) = self.left {
                ans = min_opt(ans, left.query(l, mid, x));
            }
        } else {
            if let Some(ref right) = self.right {
                ans = min_opt(ans, right.query(mid, r, x));
            }
        }
        ans
    }
}
```

## When to Reach for These

| Recurrence pattern | Optimization | Complexity reduction |
|-------------------|-------------|---------------------|
| `dp[i][j] = min(dp[i][k] + dp[k+1][j]) + C[i][j]`, C satisfies QI | Knuth | O(n³) → O(n²) |
| `dp[t][i] = min(dp[t-1][j] + cost[j][i])`, monotone opt | D&C DP | O(Tn²) → O(Tn log n) |
| `dp[i] = min(dp[j] + m[j]*x[i] + b[j])`, monotone m/x | Convex Hull | O(n²) → O(n) |
| Same, non-monotone | Li Chao Tree | O(n²) → O(n log C) |
| `dp[mask]` transition via submask enumeration | SOS DP | O(3ⁿ) → O(n·2ⁿ) |

These optimizations are worth learning because they unlock problems that are otherwise intractable. A 1000× algorithmic speedup beats any amount of SIMD tuning. But once you've applied the right algorithmic optimization, the cache-efficiency techniques from the previous article still apply — the two compound.
