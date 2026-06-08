# PageRank and Sparse Matrix-Vector Multiplication

PageRank reduces to repeated sparse matrix-vector multiplication (SpMV): multiply the adjacency matrix (normalized) by the rank vector, repeatedly, until convergence. SpMV is the most important kernel in graph analytics — it's also the kernel where memory layout matters most.

## PageRank Formulation

Given a directed graph with adjacency matrix A (A[i][j] = 1/out_degree(j) if edge j→i exists), and damping factor d (typically 0.85):

```
r_{k+1} = d · A · r_k + (1 - d) / n · 1
```

The vector `(1-d)/n · 1` is a constant. The expensive part is `A · r_k`: multiply a sparse matrix by a dense vector.

## CSR SpMV

With CSR representation, SpMV is a simple loop:

```rust
fn spmv_csr(csr: &WeightedCSR, x: &[f32], y: &mut [f32]) {
    for u in 0..csr.n {
        let start = csr.offsets[u];
        let end = csr.offsets[u + 1];
        let mut sum = 0.0f32;
        for i in start..end {
            sum += csr.weights[i] * x[csr.edges[i]];
        }
        y[u] = sum;
    }
}
```

This reads the entire CSR structure sequentially — `offsets`, `edges`, and `weights` are contiguous arrays. The random access is into `x[csr.edges[i]]` — each iteration accesses a potentially random position in `x`.

Benchmark on Zen 2 (10⁶ vertices, 10⁷ edges): ~55 ms per SpMV iteration. Memory traffic: read `x` (4 MB), read `edges` (40 MB), read `weights` (40 MB), write `y` (4 MB) = ~88 MB. At 55 ms, that's 1.6 GB/s — far below the 40 GB/s L3 cache bandwidth. The bottleneck is the random access into `x`.

## Stage 1: Cache-Aware Reordering

If we reorder the vertices so that vertices with similar neighbor sets are close together, the `x` accesses become more local. This is the **Reverse Cuthill-McKee** (RCM) ordering or **SlashBurn** for power-law graphs:

```rust
fn reorder_csr(csr: &WeightedCSR, permutation: &[usize]) -> WeightedCSR {
    // Create inverse permutation
    let mut inv_perm = vec![0usize; csr.n];
    for (i, &p) in permutation.iter().enumerate() { inv_perm[p] = i; }
    // Reorder vertices and edges
    // ...
}
```

RCM reordering on a web graph improves SpMV performance by ~1.3× (42 ms vs. 55 ms). SlashBurn (designed for power-law graphs) achieves ~1.6× (34 ms). The gain comes from fewer L3 cache misses on `x`.

## Stage 2: Tiled SpMV (Cache Blocking)

Divide the matrix into row blocks that fit in L2 cache. Process one block at a time, keeping the relevant portion of `y` in cache:

```rust
fn spmv_tiled(csr: &WeightedCSR, x: &[f32], y: &mut [f32], block_size: usize) {
    let n = csr.n;
    for block_start in (0..n).step_by(block_size) {
        let block_end = (block_start + block_size).min(n);
        for u in block_start..block_end {
            let start = csr.offsets[u];
            let end = csr.offsets[u + 1];
            let mut sum = 0.0f32;
            for i in start..end {
                sum += csr.weights[i] * x[csr.edges[i]];
            }
            y[u] = sum;
        }
    }
}
```

This helps when `y` doesn't fit in cache. For n = 10⁶, `y` is 4 MB — it fits in L3 (8 MB on Zen 2) but not L2 (512 KB). With block_size = 64K rows (256 KB of `y`), each block's output fits in L2. ~1.15× speedup.

## Stage 3: SIMD SpMV

The inner loop is a dot product between a sparse row of weights and the corresponding entries of `x`. Can we vectorize it? The challenge: `x[csr.edges[i]]` is a gather — each iteration loads from a different, unpredictable position. AVX2 has `_mm256_i32gather_ps` which does 8 gathers in one instruction:

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

unsafe fn spmv_simd(csr: &WeightedCSR, x: &[f32], y: &mut [f32]) {
    for u in 0..csr.n {
        let start = csr.offsets[u];
        let end = csr.offsets[u + 1];
        let len = end - start;
        let mut sum = _mm256_setzero_ps();
        let mut i = start;
        // Process 8 elements at a time
        while i + 8 <= end {
            let indices = _mm256_loadu_si256(
                csr.edges[i..i+8].as_ptr() as *const __m256i
            );
            let vals = _mm256_loadu_ps(csr.weights[i..i+8].as_ptr());
            let x_vals = _mm256_i32gather_ps(
                x.as_ptr(), indices, 4 /* scale */,
            );
            sum = _mm256_fmadd_ps(vals, x_vals, sum);
            i += 8;
        }
        // Horizontal sum + scalar remainder
        let mut result = horizontal_sum_256(sum);
        while i < end {
            result += csr.weights[i] * x[csr.edges[i]];
            i += 1;
        }
        y[u] = result;
    }
}
```

The gather instruction loads 8 floats from 8 potentially non-contiguous addresses in ~4 cycles (throughput: 1 per 2 cycles on Zen 2). Scalar code would need 8 individual loads (potentially 8 cache misses). Gather doesn't fix the cache misses — it just parallelizes the loads. If all 8 gathers hit L1, it's 8× faster. If all 8 miss L3, it's still 8 parallel misses — limited by memory-level parallelism (MLP).

For graphs where `x` fits in L3 cache, SIMD SpMV achieves ~1.8× speedup. For graphs where `x` exceeds cache, the speedup drops to ~1.2×.

## PageRank Implementation

```rust
fn pagerank(csr: &WeightedCSR, d: f32, epsilon: f32, max_iters: usize) -> Vec<f32> {
    let n = csr.n;
    let mut r = vec![1.0 / n as f32; n];
    let mut r_next = vec![0.0f32; n];
    let dangling_add = (1.0 - d) / n as f32;

    for _ in 0..max_iters {
        spmv_simd(csr, &r, &mut r_next);
        let mut diff = 0.0f32;
        for i in 0..n {
            r_next[i] = d * r_next[i] + dangling_add;
            diff += (r_next[i] - r[i]).abs();
            r[i] = r_next[i];
        }
        if diff < epsilon { break; }
    }
    r
}
```

Convergence on a web graph (10⁶ vertices, 10⁷ edges, d=0.85): ~50 iterations to ε=10⁻⁶. Total: ~1.7 seconds with SIMD SpMV, ~3.1 seconds with scalar.

## Beyond SpMV: Matrix-Free PageRank

For web-scale graphs (10⁹+ vertices), CSR storage alone is tens of gigabytes. The matrix-free approach stores only the edge list and recomputes values on the fly — trading compute for memory. But that's a story for the External Memory chapter (Chapter 8).

## Benchmark Summary (10⁶ vertices, 10⁷ edges, per SpMV iteration)

| Variant | Time | GFLOPS (effective) |
|---------|------|-------------------|
| CSR scalar | 55 ms | 0.36 |
| CSR + RCM reordering | 42 ms | 0.47 |
| CSR + SlashBurn | 34 ms | 0.58 |
| CSR + SIMD (no reorder) | 30 ms | 0.66 |
| CSR + SlashBurn + SIMD | 22 ms | 0.90 |

The "effective GFLOPS" is 2 × m / time — each edge contributes one multiply-add (2 FLOPs). Peak Zen 2 is 64 GFLOPS (single-precision, single core). SpMV achieves <2% of peak — it's catastrophically memory-bound. The only real lever is reducing memory traffic (reordering) or increasing parallelism (SIMD gathers).
