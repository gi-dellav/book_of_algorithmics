# Graph Representations

The representation you choose determines your memory access pattern. The access pattern determines your cache miss rate. The cache miss rate determines your runtime. Choose wrong and you lose a factor of 10 before you write a single line of algorithm code.

## Adjacency Matrix

An `n × n` matrix where `A[i][j] = 1` if edge `(i,j)` exists:

```rust
struct AdjMatrix {
    n: usize,
    data: Vec<u8>,   // or Vec<bool>, or bit-packed
}
```

Edge query `(i,j)` is O(1): `data[i * n + j]`. Memory: O(n²). For n = 10⁶, that's 10¹² entries — a terabyte, even with bits. Adjacency matrices are only viable for dense graphs (average degree > n/10) or very small n (n < 10⁴).

**When to use**: Floyd-Warshall all-pairs shortest paths, dense subgraph detection, matrix-based PageRank.

## Adjacency List (`Vec<Vec<usize>>`)

The standard textbook representation:

```rust
struct AdjList {
    edges: Vec<Vec<usize>>,  // edges[u] = neighbors of u
}
```

Memory: O(n + m), where m = number of edges. This is reasonable. The problem is layout: each `Vec<usize>` is a separate heap allocation. The neighbor lists of vertices 0, 1, 2 may be scattered across different virtual memory pages. Iterating over all edges means chasing n + 1 pointers — one per vertex's allocation, plus one for the outer Vec. On a graph with 10⁶ vertices, that's 10⁶ potential TLB misses just to access the neighbor lists.

Benchmark on Zen 2: iterating over all edges of a 10⁶-vertex, 10⁷-edge graph with `Vec<Vec<usize>>` takes ~85 ms. The same operation on CSR takes ~8 ms. The difference is entirely memory layout.

## Compressed Sparse Row (CSR)

The workhorse of high-performance graph processing. Two arrays:

```rust
struct CSR {
    n: usize,             // number of vertices
    offsets: Vec<usize>,  // offsets[u] = start of vertex u's neighbors in edges
    edges: Vec<usize>,    // concatenated neighbor lists
}
```

All neighbor lists are stored contiguously in `edges`. To iterate over vertex `u`'s neighbors:

```rust
let start = offsets[u];
let end = offsets[u + 1];
for i in start..end {
    let v = edges[i];
    // process neighbor v
}
```

One contiguous slice. The CPU's hardware prefetcher sees sequential access and prefetches the next cache lines. The TLB has one entry per 4 KB page — covering 512 `usize` entries. A vertex with degree 100 fits in one cache line (8 × 64 = 512 bytes, or 64 × 8-byte `usize` values).

## Building CSR

```rust
fn build_csr(n: usize, edge_list: &[(usize, usize)]) -> CSR {
    let m = edge_list.len();
    let mut offsets = vec![0usize; n + 1];
    let mut edges = vec![0usize; 2 * m]; // undirected: store both directions

    // Count degree of each vertex
    for &(u, v) in edge_list {
        offsets[u] += 1;
        offsets[v] += 1;
    }

    // Prefix sum → starting positions
    let mut sum = 0;
    for i in 0..n {
        let deg = offsets[i];
        offsets[i] = sum;
        sum += deg;
    }
    offsets[n] = sum;

    // Scatter edges
    let mut cursor = offsets[..n].to_vec();
    for &(u, v) in edge_list {
        edges[cursor[u]] = v;
        cursor[u] += 1;
        edges[cursor[v]] = u;
        cursor[v] += 1;
    }

    CSR { n, offsets, edges }
}
```

O(n + m) time, O(n + m) memory. The prefix sum pass is trivially vectorizable. The scatter pass has random writes into `edges`, but `cursor[i]` stays in L1 cache. Overall throughput: ~200 million edges/second on Zen 2.

## CSR Variants

### CSR with Weights

Add a `weights: Vec<f32>` array parallel to `edges`, storing edge weights:

```rust
struct WeightedCSR {
    n: usize,
    offsets: Vec<usize>,
    edges: Vec<usize>,
    weights: Vec<f32>,
}
```

### CSR for Directed Graphs

Only store outgoing edges. Add `in_offsets` and `in_edges` if you need incoming edges (e.g., for PageRank). The cost: 2× the edges storage.

### COO (Coordinate Format)

Store edges as three parallel arrays: `row: Vec<usize>`, `col: Vec<usize>`, `val: Vec<f32>`. Good for construction (append-only), terrible for traversal (random access into all three). Convert to CSR before running algorithms.

### DCSC (Doubly-Compressed Sparse Column)

Like CSR but for hypersparse matrices (graphs where most vertices have degree 0 or 1). Compresses the `offsets` array by only storing non-empty columns. Used in column-store databases.

## When to Use What

| Representation | Edge query | Iterate neighbors | Memory | Best for |
|---------------|-----------|-------------------|--------|----------|
| Adjacency matrix | O(1) | O(n) | O(n²) | Dense graphs, Floyd-Warshall |
| `Vec<Vec<usize>>` | O(1) | O(deg) | O(n + m) | Prototyping, dynamic graphs |
| CSR | O(1) | O(deg) contiguous | O(n + m) | Production graph processing |
| COO | O(m) scan | O(m) scan | O(m) | Construction, conversion |

## The 10× Factor

Returning to the benchmark: why is CSR 10× faster than `Vec<Vec<usize>>` for a full edge iteration?

- **Prefetching**: CSR's contiguous `edges` array enables hardware prefetching. `Vec<Vec<>>`'s scattered allocations defeat it.
- **TLB**: CSR touches n + (m × 8 / 4096) pages. `Vec<Vec<>>` touches n + n × (deg × 8 / 4096) pages — each vertex's allocation may cross a page boundary.
- **Cache line utilization**: CSR packs 8 neighbors per cache line (64 bytes / 8 bytes). `Vec<Vec<>>` has per-allocation overhead (the Vec's capacity and pointer), wasting the first 16–24 bytes of each allocation.

The lesson: graph algorithms live and die by memory layout. Always convert to CSR before benchmarking.
