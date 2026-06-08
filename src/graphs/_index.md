# The Hidden Cost of Pointer Chasing

Graph algorithms are the canonical example of where Big O analysis fails catastrophically on real hardware. A BFS that visits vertices in the "wrong" order can be 100× slower than one that respects cache topology — despite both doing exactly O(V + E) work. The reason: graphs are sparse in memory the same way they are sparse in edges, and every random vertex access is a potential cache miss.

## The Graph Memory Problem

A graph is a set of vertices connected by edges. In memory, this becomes pointers. Lots of pointers. An adjacency list representation stores, for each vertex, a list (or vector) of neighbor indices. A BFS traversal follows these pointers:

```
for each neighbor v of current vertex u:
    if not visited[v]:
        visited[v] = true
        queue.push(v)
```

Every access to `visited[v]` and every dereference into the adjacency list is potentially a cache miss. For a graph with 10⁶ vertices and average degree 10, a BFS might incur a million cache misses. That's ~100 million cycles — 50 milliseconds — just waiting for memory. On the same CPU, 100 million cycles could execute ~400 million integer additions.

And that's just BFS. Dijkstra's algorithm adds a priority queue. Betweenness centrality runs BFS from every vertex. PageRank iterates until convergence. Graph algorithms are memory-bound almost by definition.

## What This Chapter Covers

1. **Graph Representations** — Adjacency matrix vs. adjacency list vs. CSR (Compressed Sparse Row). The 10× performance difference between `Vec<Vec<usize>>` and CSR, explained.
2. **BFS and Dijkstra** — Direction-optimizing BFS (Beamer, Asanović, Patterson 2012). `std::collections::BinaryHeap` vs. radix-heap vs. bucket queue for Dijkstra on integer weights.
3. **Connected Components** — Union-find with path compression, the Afrand optimizations, and when to use BFS instead.
4. **Minimum Spanning Tree** — Kruskal vs. Prim. Why Kruskal usually wins (sorting is cache-friendly; priority queues are not).
5. **PageRank and Iterative Solvers** — Sparse matrix-vector multiplication (SpMV) as the computational kernel. Cache blocking for power-law graphs.

## Recommended Reading Order

Read **Graph Representations** first — every other article depends on understanding CSR layout and why it matters. Then BFS/Dijkstra, then the rest in any order. Cross-reference with Chapter 12 (SIMD) for SpMV vectorization and Chapter 8 (External Memory) for cache-oblivious graph algorithms.
