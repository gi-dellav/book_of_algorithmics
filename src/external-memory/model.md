# The External Memory Model

The External Memory model (Aggarwal and Vitter, 1988) captures the cost of data movement between two levels of the memory hierarchy. It's the most useful theoretical model for reasoning about cache performance.

## Parameters

- **N**: Problem size (number of elements).
- **M**: Internal memory size (number of elements that fit in the fast memory).
- **B**: Block size (number of elements transferred in one I/O operation).

Modern mapping: M = cache size, B = cache line size (64 bytes = 8 doubles = 16 ints).

## Cost Model

- Computation on data in internal memory: **free**.
- Moving a block between external and internal memory: **1 I/O**.
- The algorithm explicitly manages which blocks are in internal memory.

The goal: minimize the number of I/Os, subject to the constraint that at most M elements can be in internal memory at any time.

Two fundamental operations:
- **SCAN(N)**: Read N elements sequentially. Cost: ⌈N/B⌉ I/Os.
- **SORT(N)**: Sort N elements. Cost: Θ((N/B) × log_{M/B}(N/B)) I/Os (achievable with multiway merge sort).

## Why This Model Works

The model correctly predicts that:
- **Scanning is cheap**: Reading N elements costs N/B I/Os — much less than N when B is large. A sequential scan of 1 MB costs ~16,000 cache line transfers (1 MB / 64 bytes), not 250,000 int-sized reads.
- **Sorting is more expensive than scanning but cheaper than random access**: The log_{M/B} factor is usually small (M/B is thousands to millions, so log_{M/B}(N/B) is 1–3 for realistic problem sizes).
- **Random access is devastating**: Each random access may cost 1 I/O (if the block isn't already in cache). N random accesses = N I/Os.

## The Tall Cache Assumption

Most theoretical analyses assume M ≥ B² (the cache is "taller than it is wide"). This holds for real caches:
- L1: 32 KB = 512 cache lines of 64 B. 512 ≫ 64, so M = 512B, B² = 4096. Not satisfied! (512 < 4096)
- L2: 512 KB = 8192 cache lines. 8192 ≫ 64, so M ≈ 128B². Satisfied.
- RAM-to-disk: M = gigabytes, B = 4 KB pages. M ≫ B². Satisfied.

When the tall cache assumption fails (L1), the asymptotic bounds still hold but with slightly larger constants. For practical optimization, use the specific cache parameters from Chapter 9 rather than asymptotic bounds.

## Example: Matrix Transposition

Transpose an N × N matrix in the external memory model.

**Naive**: for i, j: swap A[i][j] and A[j][i].

If the matrix is stored in row-major order, accessing A[j][i] for each j (across rows) reads one element from a different cache line each time. I/O cost: Θ(N²) — essentially every access is a miss.

**Blocked**: Divide into B × B blocks. Transpose each block in internal memory (free), then write back. Each block contributes 2B²/B = 2B I/Os. Total: Θ(N²/B) — a factor of B improvement.

**Cache-oblivious** (divide and conquer): Recursively divide the matrix into quadrants until they fit in cache. The same asymptotic I/O cost Θ(N²/B) is achieved *without knowing B*. The recursion automatically adapts to any cache size. This is the subject of `oblivious.md`.

## Limitations of the Model

1. **Replacement policy**: The model assumes the algorithm controls eviction (like an optimal offline policy). Real caches use LRU or variants. The Sleator-Tarjan theorem (see `policies.md`) shows LRU is within a factor of 2 of optimal for most algorithms.
2. **No prefetching**: The model doesn't capture hardware prefetchers, which can detect sequential patterns and fetch ahead automatically.
3. **Constant factors**: The asymptotic analysis hides constant factors that dominate for small N.
4. **Multiple levels**: The model has two levels (fast/slow). Real hierarchies have 4–5 levels. The same analysis applies recursively.
5. **Write-back caching**: The model counts reads and writes equally. Real caches may have write-allocate policies that add extra I/Os on writes.

Despite these limitations, the external memory model is the single most useful theoretical tool for understanding cache performance. Algorithms designed for it (blocked algorithms, cache-oblivious algorithms, B-trees) are the ones that perform well on real hardware.
