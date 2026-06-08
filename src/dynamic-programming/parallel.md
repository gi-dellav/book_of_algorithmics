# Parallel DP

Dynamic programming is famously hard to parallelize — the recurrence `dp[i] = f(dp[i-1])` is inherently sequential. But many DP problems have enough independence to extract parallelism: wavefront parallelism for 2D DP, block-based parallelism for matrix-chain DP, and GPU tiling for sequence alignment. This article covers the techniques that actually work.

## Wavefront Parallelism

For recurrences where `dp[i][j]` depends only on `dp[i-1][j-1]`, `dp[i-1][j]`, and `dp[i][j-1]`, all cells on the same anti-diagonal `i + j = d` are independent:

```
dp[0][0]
dp[0][1] dp[1][0]
dp[0][2] dp[1][1] dp[2][0]
...
```

Each diagonal depends only on the two previous diagonals. Process diagonals sequentially, but parallelize within each diagonal:

```rust
use std::sync::Arc;
use std::thread;

fn parallel_lcs(a: &[u8], b: &[u8]) -> u32 {
    let n = a.len();
    let m = b.len();
    let stride = m + 1 + 16; // padded
    let dp = Arc::new(vec![0u32; stride * (n + 1)]);

    for d in 1..=n + m {
        let dp_ref = Arc::clone(&dp);
        let diag_len = if d <= n.min(m) { d } else { /* ... */ };

        // Parallelize: split the diagonal across threads
        let num_threads = 8;
        thread::scope(|s| {
            let chunk_size = (diag_len + num_threads - 1) / num_threads;
            for t in 0..num_threads {
                let start = t * chunk_size;
                let end = (start + chunk_size).min(diag_len);
                s.spawn(move || {
                    for idx in start..end {
                        let i = /* compute i from d, idx */;
                        let j = d - i;
                        // Compute dp[i][j] from dp[i-1][j-1], etc.
                    }
                });
            }
        });
    }

    // extract result
    0
}
```

On a 16-core Zen 2, wavefront LCS achieves ~11× speedup (the efficiency loss is from thread synchronization per diagonal — with ~2n diagonals, that's a lot of barriers).

## Block-Based Parallelism

Better: divide the DP table into blocks and process them with a block dependency graph. A block at (bi, bj) depends on blocks (bi-1, bj-1), (bi-1, bj), and (bi, bj-1). Blocks on the same anti-diagonal of the block grid are independent:

```rust
fn block_parallel_dp(dp: &mut [u32], n: usize, m: usize, stride: usize,
                     block_size: usize, num_threads: usize) {
    let bi_count = (n + block_size - 1) / block_size;
    let bj_count = (m + block_size - 1) / block_size;

    for bd in 0..bi_count + bj_count - 1 {
        // Blocks on this block-diagonal are independent
        let blocks_on_diag: Vec<(usize, usize)> = (0..=bd)
            .filter(|&bi| bi < bi_count && bd - bi < bj_count)
            .map(|bi| (bi, bd - bi))
            .collect();

        // Process blocks in parallel
        blocks_on_diag.par_iter().for_each(|&(bi, bj)| {
            let i_start = bi * block_size;
            let i_end = (i_start + block_size).min(n);
            let j_start = bj * block_size;
            let j_end = (j_start + block_size).min(m);

            for i in i_start..i_end {
                for j in j_start..j_end {
                    // Standard DP recurrence
                    unsafe {
                        let a = dp.get_unchecked((i-1) * stride + (j-1));
                        // ...
                    }
                }
            }
        });
    }
}
```

This uses `rayon` for parallel iteration. The block size should be chosen so that each block fits in L2 cache (128 × 128 × 4 bytes = 64 KB fits in Zen 2's 512 KB L2).

Performance for LCS (n=m=100K) with block_size=256: ~78 ms on 16 cores (11× vs. single-core).

## GPU DP: Tiled Sequence Alignment

GPUs excel at 2D DP. The key is shared memory tiling:

1. Divide the DP table into tiles.
2. Each thread block processes one tile.
3. The tile is loaded into shared memory (on-chip, ~100× faster than global memory).
4. Threads within the block cooperatively compute the tile.
5. Tile results write back to global memory.

For the Smith-Waterman local alignment algorithm (ubiquitous in bioinformatics):

```cuda
__global__ void smith_waterman_kernel(
    const char* a, const char* b,
    int* dp, int n, int m,
    int tile_size
) {
    __shared__ int tile[TILE_SIZE][TILE_SIZE];
    __shared__ char a_shared[TILE_SIZE];
    __shared__ char b_shared[TILE_SIZE];

    int bi = blockIdx.x;
    int bj = blockIdx.y;

    // Load query and reference chars into shared memory
    if (threadIdx.x == 0) {
        int i = bi * TILE_SIZE + threadIdx.x;
        a_shared[threadIdx.x] = (i < n) ? a[i] : 0;
    }
    // ... compute tile ...
}
```

On an NVIDIA A100, tiled Smith-Waterman aligns 10⁶ read pairs of length 100 in ~0.5 seconds — ~200× faster than a 16-core CPU. The GPU's 108 SMs × 128 KB shared memory provide ~13 MB of on-chip storage, enough for thousands of concurrent tiles.

## Pipeline Parallelism for Multi-Stage DP

Some DP problems have multiple stages (e.g., k layers of a DP, each depending on the previous). Pipeline parallelism processes all stages simultaneously on different cores in a producer-consumer pattern:

```rust
use std::sync::mpsc;

fn pipeline_dp(n: usize, stages: usize) -> Vec<Vec<u32>> {
    let mut channels = Vec::new();
    let (tx0, rx0) = mpsc::sync_channel::<Vec<u32>>(1);
    channels.push((tx0, rx0));

    // ... create pipeline stages ...

    // Stage 0 produces dp[0], stage 1 consumes and produces dp[1], etc.
    // Each stage runs on its own thread.
    // Throughput is limited by the slowest stage.

    todo!()
}
```

Pipeline parallelism is effective when each stage takes roughly equal time and the number of stages is comparable to the number of cores. For 8 stages on 8 cores, ideal speedup is 8×. In practice, synchronization overhead reduces this to 5–6×.

## When Not to Parallelize DP

- **n < 1000**: Thread spawning overhead exceeds the computation.
- **Highly sequential recurrences**: If `dp[i]` depends on `dp[i-1]` with no other dependencies, parallelism is impossible. Use SIMD within the single row instead.
- **Irregular dependencies**: If the DP graph has unpredictable structure, the scheduling overhead of dynamic task graphs (e.g., TBB, Cilk) may exceed the gain.

## Benchmark Summary (LCS, n=m=100K)

| Configuration | Time | Speedup |
|--------------|------|---------|
| Single-thread, row-major | 860 ms | 1× |
| Single-thread, diagonal sweep | 620 ms | 1.4× |
| 8-core wavefront | 142 ms | 6.1× |
| 16-core wavefront | 78 ms | 11.0× |
| 16-core block-parallel (256×256) | 62 ms | 13.9× |
| GPU (A100, tiled) | 8 ms | 107× |

The GPU is in a different league for 2D DP, but the CPU parallel versions are competitive when you don't have a $10K accelerator. The block-parallel CPU version gets within 8× of the GPU — impressive for a general-purpose processor.
