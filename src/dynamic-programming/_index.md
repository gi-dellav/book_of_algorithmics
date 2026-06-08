# Tables, Not Recursion

Dynamic programming is the algorithmic equivalent of memoization: break a problem into overlapping subproblems, solve each subproblem once, and store the results in a table. The textbook teaches DP as recursion + memoization. Hardware teaches that the table's layout in memory determines whether your DP runs in seconds or minutes.

## The DP Memory Wall

A typical DP recurrence accesses `dp[i-1][j]`, `dp[i][j-1]`, and `dp[i-1][j-1]`. If the table is stored in row-major order (the default for C/Rust 2D arrays), `dp[i][j-1]` is adjacent in memory (sequential access, cache-friendly). `dp[i-1][j]` is one row back — for a 10,000 × 10,000 table, that's 40 KB back (contiguous with the previous row, cache-friendly if the row fits in L1). `dp[i-1][j-1]` is the same.

But if you store the table in column-major order and access by row, every `dp[i-1][j]` is 10,000 elements apart — a cache miss on every access. The same DP algorithm can be 50× slower with the wrong memory layout.

And that's just the simplest case. DP on implicit graphs (shortest paths in a DAG), DP with monotonicity optimizations (Knuth optimization, divide-and-conquer DP), and DP with bitmask states (for combinatorial problems) each have their own cache and parallelism considerations.

## What This Chapter Covers

1. **Cache-Efficient DP** — Loop ordering for 2D DP tables. The `ikj` vs `ijk` ordering for DP on matrices. Memory layout for DP over sequences.
2. **DP Optimizations** — Knuth optimization (quadrangle inequality), divide-and-conquer DP, convex hull trick, Li Chao tree. When O(n³) becomes O(n²) or O(n log n) with monotonicity.
3. **Bitmask DP** — Held-Karp TSP, subset convolution with SIMD, bit-parallel DP for small state spaces.
4. **Parallel DP** — Wavefront parallelism, block-based parallelism for 2D DP, GPU DP with shared memory tiling.

## Recommended Reading Order

Start with **Cache-Efficient DP** — it's the foundation for everything else. Then DP Optimizations for the algorithmic speedups. Bitmask DP is self-contained. Parallel DP ties together the parallel computing chapter (Chapter 13) with DP.

Each article assumes you know what DP is (overlapping subproblems, optimal substructure, Bellman optimality). If you need a refresher, any algorithms textbook will do — this chapter is about making DP *fast*, not about teaching it from scratch.
