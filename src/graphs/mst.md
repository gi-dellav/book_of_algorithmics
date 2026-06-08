# Minimum Spanning Tree

Given a connected, undirected, weighted graph, find the subset of edges that connects all vertices with minimum total weight. Two classic algorithms — Kruskal and Prim — both run in O(m log n) time, but on real hardware Kruskal is usually 3–5× faster. The reason reveals a deep principle: sorting is cache-friendly; priority queues are not.

## Kruskal's Algorithm

Sort all edges by weight, then iterate, adding each edge to the MST if it connects two different components. Uses union-find for component tracking:

```rust
fn kruskal(n: usize, edges: &mut [(usize, usize, f32)]) -> Vec<(usize, usize, f32)> {
    edges.sort_unstable_by(|a, b| a.2.partial_cmp(&b.2).unwrap());
    let mut dsu = DSU::new(n);
    let mut mst = Vec::with_capacity(n - 1);
    for &(u, v, w) in edges.iter() {
        if dsu.find(u) != dsu.find(v) {
            dsu.union(u, v);
            mst.push((u, v, w));
            if mst.len() == n - 1 { break; }
        }
    }
    mst
}
```

The sort dominates runtime: O(m log m) comparisons, but comparisons are cheap and the sort is a sequential scan after the O(m log m) divide-and-conquer passes. Rust's `sort_unstable` (pdqsort) achieves ~200M comparisons/second on `f32` keys.

The union-find pass is O(m α(n)) with random memory accesses, but it's only ~15% of total runtime because `m` is usually small relative to the sort time.

## Prim's Algorithm

Start from an arbitrary vertex, repeatedly add the cheapest edge connecting the current tree to a new vertex:

```rust
fn prim(csr: &WeightedCSR, start: usize) -> Vec<(usize, usize, f32)> {
    let n = csr.n;
    let mut in_tree = vec![false; n];
    let mut best_edge = vec![(usize::MAX, f32::INFINITY); n];
    let mut pq = std::collections::BinaryHeap::new();
    let mut mst = Vec::with_capacity(n - 1);

    in_tree[start] = true;
    let s = csr.offsets[start];
    let e = csr.offsets[start + 1];
    for i in s..e {
        let v = csr.edges[i];
        let w = csr.weights[i];
        best_edge[v] = (start, w);
        pq.push(State { cost: std::cmp::Reverse(OrderedFloat(w)), vertex: v });
    }

    while let Some(State { cost: _, vertex: u }) = pq.pop() {
        if in_tree[u] { continue; }
        in_tree[u] = true;
        let (p, _) = best_edge[u];
        mst.push((p, u, /* weight */));

        let s = csr.offsets[u];
        let e = csr.offsets[u + 1];
        for i in s..e {
            let v = csr.edges[i];
            let w = csr.weights[i];
            if !in_tree[v] && w < best_edge[v].1 {
                best_edge[v] = (u, w);
                pq.push(State { cost: std::cmp::Reverse(OrderedFloat(w)), vertex: v });
            }
        }
    }
    mst
}
```

The priority queue is the bottleneck: `BinaryHeap::push` and `pop` are O(log n) each, with log n typically 15–25. Each operation touches 2–3 cache lines in the heap array. For a dense graph (m ≈ n²), the heap has O(n) entries and processes O(m) updates — that's m × log n random heap accesses.

## Why Kruskal Wins

| Aspect | Kruskal | Prim |
|--------|---------|------|
| Sorting | O(m log m) sequential | — |
| Priority queue | — | O(m log n) random |
| Memory access | Streaming through sorted edges | Random heap accesses |
| Union-find | O(m α(n)) random, but small constant | — |
| SIMD-friendly | Sort can be vectorized | Heap operations cannot |

The fundamental asymmetry: sorting is the most optimized operation in computing. Libraries (pdqsort, vqsort, radix sort) achieve near-bandwidth-limited performance. Priority queues have poor cache locality and no SIMD path. Kruskal routes the work through the fast path; Prim through the slow one.

Benchmark on Zen 2 (10⁵ vertices, 5×10⁵ edges, random weights):

| Algorithm | Time |
|-----------|------|
| Kruskal (pdqsort + packed DSU) | 28 ms |
| Prim (`BinaryHeap`) | 94 ms |
| Prim (radix heap, integer weights) | 45 ms |

## When Prim Wins

Prim can beat Kruskal on dense graphs (m ≈ n²) because:
1. Kruskal sorts all m edges — O(n²) — while Prim's heap has at most n entries.
2. Prim can stop after n-1 edges are added; Kruskal must examine all edges (or sort them).
3. Prim naturally produces the MST rooted at a specific vertex, which is useful for some applications (e.g., hierarchical clustering).

The crossover is around m/n > 100 (average degree > 100). Below that, Kruskal is faster. Above, Prim catches up. But most real-world graphs are sparse (average degree < 50), so Kruskal is the default choice.

## Borůvka's Algorithm

A third contender: Borůvka's algorithm adds the cheapest edge from each component to any other component, in parallel, halving the number of components each round. O(m log n) time, but highly parallel:

```rust
fn boruvka(n: usize, edges: &[(usize, usize, f32)]) -> Vec<(usize, usize, f32)> {
    let mut dsu = DSU::new(n);
    let mut components = n;
    let mut mst = Vec::with_capacity(n - 1);

    while components > 1 {
        // Find cheapest outgoing edge for each component
        let mut cheapest = vec![(usize::MAX, f32::INFINITY); n];
        for &(u, v, w) in edges.iter() {
            let ru = dsu.find(u);
            let rv = dsu.find(v);
            if ru != rv {
                if w < cheapest[ru].1 { cheapest[ru] = (v, w); }
                if w < cheapest[rv].1 { cheapest[rv] = (u, w); }
            }
        }
        // Add cheapest edges
        for u in 0..n {
            if cheapest[u].0 != usize::MAX {
                let v = cheapest[u].0;
                if dsu.find(u) != dsu.find(v) {
                    dsu.union(u, v);
                    mst.push((u, v, cheapest[u].1));
                    components -= 1;
                }
            }
        }
    }
    mst
}
```

Borůvka shines on GPUs: each round, all edges can be processed in parallel. On a Galaxy-class graph (10⁹ edges), Borůvka on an NVIDIA A100 finishes in ~200 ms — competitive with the best CPU algorithms, despite the GPU's lower clock.

## The Deeper Lesson

Kruskal vs. Prim is not about asymptotic complexity (both are O(m log n)). It's about which operations the algorithm uses. Sorting an array is fast because it's sequential and vectorizable. Heap operations are slow because they're random and scalar. When designing algorithms, prefer operations that hardware is good at: sorting, scanning, partitioning. Avoid pointer chasing and random heap access. This principle recurs throughout this chapter — and this book.
