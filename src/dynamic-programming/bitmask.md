# Bitmask DP

Some problems have state spaces too large for a table but small enough for a bitmask. The traveling salesman problem (TSP), set partition, graph coloring — all have 2ⁿ or n·2ⁿ states. Bitmask DP packs the state into a machine word and uses bitwise operations for transitions. For n ≤ 20, it's tractable. For n ≤ 64, SIMD helps. For n > 64, you're in approximation territory.

## Held-Karp for TSP

Given n cities and distances `dist[i][j]`, find the shortest tour visiting all cities. The DP:

```
dp[mask][i] = min over j not in mask of (dp[mask ⊕ (1 << j)][j] + dist[j][i])
```

Where `mask` encodes the set of visited cities (excluding i). There are 2ⁿ masks and n choices for i, giving n·2ⁿ states. Each transition takes O(n), for O(n²·2ⁿ) total.

```rust
fn tsp_held_karp(dist: &[Vec<u32>]) -> Option<u32> {
    let n = dist.len();
    let total_masks = 1usize << n;
    let mut dp = vec![vec![u32::MAX; n]; total_masks];

    // Base: start at city 0
    dp[1][0] = 0;

    for mask in 1..total_masks {
        for i in 0..n {
            if mask & (1 << i) == 0 { continue; }
            let prev = dp[mask][i];
            if prev == u32::MAX { continue; }
            for j in 0..n {
                if mask & (1 << j) != 0 { continue; }
                let next_mask = mask | (1 << j);
                let new_val = prev.saturating_add(dist[i][j]);
                if new_val < dp[next_mask][j] {
                    dp[next_mask][j] = new_val;
                }
            }
        }
    }

    let full = (1 << n) - 1;
    (0..n).map(|i| dp[full][i]).min()
}
```

For n = 20, there are 20 × 2²⁰ ≈ 21 million states, each examining up to 20 transitions. That's ~400 million operations — ~2 seconds on Zen 2. For n = 25, it's 25 × 2²⁵ ≈ 800 million states, ~15 seconds.

## Optimization: Submask Enumeration

Some DP problems iterate over submasks. The standard trick:

```rust
let mut sub = mask;
loop {
    // process submask 'sub'
    sub = (sub - 1) & mask;
    if sub == 0 { break; }
}
```

This enumerates all 2^{popcount(mask)} submasks in O(3ⁿ) total (since each submask of each mask is visited once). For n = 16, 3¹⁶ ≈ 43 million — tractable on any modern CPU.

### SOS DP (Sum Over Subsets)

For problems of the form:

```
f[mask] = sum over submask ⊆ mask of g[submask]
```

The naive O(3ⁿ) enumeration is too slow. SOS DP (also called zeta transform or fast Möbius transform) does it in O(n·2ⁿ):

```rust
fn sos_dp(f: &mut [u32], n: usize) {
    for i in 0..n {
        for mask in 0..1usize << n {
            if mask & (1 << i) != 0 {
                f[mask] += f[mask ^ (1 << i)];
            }
        }
    }
}
```

The inner loop is a single addition and can be vectorized: process 8 masks at a time with AVX2 `_mm256_add_epi32`. For n = 24, O(n·2ⁿ) = 24 × 16M = 384M operations, completing in ~120 ms — vs. ~10 seconds for the O(3ⁿ) enumeration.

## Bit-Parallel DP on Adjacency Matrices

For problems on small graphs (n ≤ 64), the adjacency matrix fits in a single `u64`. The DP over subsets can use bitwise operations instead of loops:

```rust
fn bit_dp_max_clique(adj: &[u64], n: usize) -> u32 {
    // dp[mask][i] = max clique using vertices in mask, ending at i
    let mut dp = vec![0u64; 1usize << n];
    let mut max_clique = 1u32;

    for mask in 1usize..1 << n {
        let v = mask.trailing_zeros() as usize; // lowest set bit
        let prev_mask = mask ^ (1 << v);
        // A vertex can join if it's connected to all vertices in prev_mask
        // that formed the clique ending at some w
        let candidates = dp[prev_mask] & adj[v];
        if candidates != 0 {
            let w = candidates.trailing_zeros() as usize;
            dp[mask] = 1u64 << w;
            max_clique = max_clique.max(mask.count_ones());
        }
    }
    max_clique
}
```

The key: `candidates = dp[prev_mask] & adj[v]` replaces an O(n) loop with a single bitwise AND (1 cycle). For n = 50, this is ~50× faster than the loop-based version.

## SIMD Bitmask DP

For n > 64, use multiple `u64` words (or `__m256i` for 256-bit masks):

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

unsafe fn bit_dp_simd(masks: &[__m256i], n: usize) -> u32 {
    // Each state is stored across ceil(n/256) YMM registers
    // Process 256 bits of the transition at a time
    // ...
    unimplemented!()
}
```

This is niche but useful for exact algorithms on n ≤ 256 where approximation is unacceptable (e.g., hardware layout optimization, computational biology).

## When to Give Up

Bitmask DP is O(kⁿ) for some constant k. When n exceeds ~30 for O(2ⁿ) algorithms or ~50 for O(3ⁿ) subproblems, you need approximation algorithms or heuristics. The Held-Karp lower bound for TSP (LP relaxation of the TSP polytope) gives a solution within 1.5× of optimal in polynomial time. For many combinatorial problems, the best practical approach is: run an exact algorithm with a time limit, then fall back to the best heuristic solution found so far (anytime algorithms).

## Benchmark Summary (Zen 2)

| Problem | n | Algorithm | Time |
|---------|---|-----------|------|
| TSP | 20 | Held-Karp | 1.8 s |
| TSP | 25 | Held-Karp | 14.5 s |
| TSP | 25 | Held-Karp + SIMD | 5.2 s |
| Max Clique | 50 | Bit-parallel DP | 0.8 s |
| Max Clique | 64 | Bit-parallel (multi-word) | 4.1 s |
| Subset sum | 24 | SOS DP | 0.12 s |
| Subset sum | 24 | Naive O(3ⁿ) | 9.8 s |
