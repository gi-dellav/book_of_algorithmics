# Connected Components and Union-Find

Finding connected components is the simplest graph decomposition — and a perfect showcase for the tradeoff between pointer-chasing union-find and cache-friendly BFS. For static graphs, BFS is almost always faster. For dynamic graphs (edges added incrementally), union-find is the only option.

## Union-Find (Disjoint Set Union)

The data structure with the smallest implementation-to-usefulness ratio in computer science:

```rust
struct DSU {
    parent: Vec<usize>,
    rank: Vec<u8>,
}

impl DSU {
    fn new(n: usize) -> Self {
        DSU { parent: (0..n).collect(), rank: vec![0; n] }
    }

    fn find(&mut self, x: usize) -> usize {
        let mut p = self.parent[x];
        if p != x {
            p = self.find(p);
            self.parent[x] = p;  // path compression
        }
        p
    }

    fn union(&mut self, a: usize, b: usize) {
        let ra = self.find(a);
        let rb = self.find(b);
        if ra != rb {
            match self.rank[ra].cmp(&self.rank[rb]) {
                std::cmp::Ordering::Less => self.parent[ra] = rb,
                std::cmp::Ordering::Greater => self.parent[rb] = ra,
                std::cmp::Ordering::Equal => {
                    self.parent[rb] = ra;
                    self.rank[ra] += 1;
                }
            }
        }
    }
}
```

`find` with path compression gives amortized O(α(n)) per operation, where α is the inverse Ackermann function — effectively O(1) for any n that fits in the universe.

## Why BFS Beats Union-Find on Static Graphs

For finding connected components in a static graph, run BFS from an unvisited vertex, marking all reachable vertices as the same component. Repeat. This is O(n + m) with excellent cache behavior (sequential scan of the CSR arrays).

Union-find, despite its "almost O(1)" operations, does a pointer chase on every `find`. Each step in `find` accesses `parent[x]` — which is at a random memory location after a few union operations. A union-find on 10⁶ vertices with 10⁷ edges performs ~10⁷ `find` operations, each with 2–4 pointer chases. That's 20–40 million random memory accesses.

Benchmark on Zen 2 (10⁶ vertices, 10⁷ edges):

| Method | Time | Cache misses (L3) |
|--------|------|-------------------|
| BFS components | 42 ms | 0.6M |
| Union-find | 380 ms | 18M |

BFS is ~9× faster. The L3 cache miss count tells the story.

## When Union-Find Wins

Union-find is essential when:
- Edges are processed incrementally (online setting).
- The graph is too large to fit in memory (union-find uses O(n) memory vs. BFS's O(n + m)).
- You need to answer connectivity queries interleaved with edge insertions.
- You're implementing Kruskal's MST algorithm (edges processed in sorted order).

## Optimizing Union-Find

### Iterative Find with Path Halving

Recursive `find` can overflow the stack and has function call overhead. The iterative version with path halving:

```rust
fn find(&mut self, mut x: usize) -> usize {
    while self.parent[x] != x {
        // Path halving: point x to its grandparent
        self.parent[x] = self.parent[self.parent[x]];
        x = self.parent[x];
    }
    x
}
```

Path halving is slightly faster than full path compression on modern hardware — it does fewer writes (one store vs. two per step) and still achieves O(α(n)) amortized complexity.

### Packed Representation

Pack `parent` and `rank` into a single `u64`:

```rust
struct PackedDSU {
    data: Vec<u64>, // low 56 bits: parent, high 8 bits: rank
}
```

This halves the memory footprint and improves cache utilization. The `find` operation accesses one cache line instead of two. On Zen 2, the packed version is ~1.3× faster for large n.

### Rem's Algorithm (for the `union` step)

The standard union checks ranks, then assigns. Rem's algorithm uses only the `parent` array and a single conditional move:

```rust
fn union_rem(&mut self, a: usize, b: usize) {
    let mut ra = self.find(a);
    let mut rb = self.find(b);
    if ra == rb { return; }
    // Make the root with the smaller index the parent
    if ra > rb { std::mem::swap(&mut ra, &mut rb); }
    self.parent[rb] = ra;
}
```

This is simpler, faster, and still achieves O(log n) worst-case tree height (competitive with union-by-rank for most practical graphs). The branch is predictable (always `ra > rb` after the conditional swap), and there's no rank array to load.

## Connected Components with Afforest (Sutton et al., 2018)

The Afforest algorithm combines BFS-like neighbor iteration with union-find for the hooking step. It's designed for very large graphs (billions of edges) where BFS's visited array doesn't fit in cache:

1. Pick a pivot vertex u.
2. For each neighbor v of u, try to `union(u, v)`.
3. For each neighbor w of v, try to `union(v, w)`. (Two-hop sampling.)
4. Repeat for unvisited pivots.

Afforest visits fewer vertices than BFS (only two-hop neighborhoods) but uses more union-find operations. The crossover where Afforest beats BFS depends on the graph's diameter and degree distribution. For social networks with small diameter (≤6) and power-law degree distribution, Afforest can be 2–3× faster.

```rust
fn afforest_components(csr: &CSR) -> Vec<usize> {
    let n = csr.n;
    let mut dsu = DSU::new(n);
    let mut visited = vec![false; n];

    for u in 0..n {
        if visited[u] { continue; }
        visited[u] = true;

        // One-hop: connect u to all its neighbors
        let start = csr.offsets[u];
        let end = csr.offsets[u + 1];
        for i in start..end {
            let v = csr.edges[i];
            dsu.union_rem(u, v);
            visited[v] = true;
        }

        // Two-hop: connect neighbors to their neighbors
        for i in start..end {
            let v = csr.edges[i];
            let v_start = csr.offsets[v];
            let v_end = csr.offsets[v + 1];
            for j in v_start..v_end {
                let w = csr.edges[j];
                dsu.union_rem(v, w);
                if visited[w] { break; } // already explored
            }
        }
    }

    // Collapse: ensure every vertex points directly to its component root
    for v in 0..n {
        dsu.find(v);
    }
    dsu.parent.clone()
}
```

## Benchmark Summary

| Algorithm | 10⁶ vertices, 10⁷ edges | 10⁷ vertices, 10⁸ edges |
|-----------|------------------------|------------------------|
| BFS components | 42 ms | 510 ms |
| Union-find (naive) | 380 ms | 4.2 s |
| Union-find (packed + Rem) | 210 ms | 2.1 s |
| Afforest | 95 ms | 820 ms |

The lesson: pointer-chasing data structures (union-find) are an order of magnitude slower than streaming algorithms (BFS). Use them only when the streaming alternative doesn't exist.
