# Distributed Algorithms

Distributed algorithms solve problems that span multiple machines connected by a network. They're not just "parallel algorithms with a network" — the network fundamentally changes the design space: communication costs dominate, failures are expected, and load imbalance is multiplied by the variance of hardware and network conditions.

## Distributed Sorting

Sorting 1 TB of integers across 100 nodes, each with 10 GB of RAM. A single-machine sort (assuming enough RAM) is O(n log n). A distributed sort must minimize network transfers while keeping nodes balanced.

### Sample Sort

The standard distributed sorting algorithm:

1. **Sample**: each node randomly samples ~1000 keys from its local data. Send samples to the coordinator (process 0).
2. **Find splitters**: the coordinator sorts all samples and picks P-1 splitters that divide the samples into P equal buckets. Broadcast splitters to all nodes.
3. **Partition**: each node scans its local data and sends each element to the node responsible for its bucket (based on splitters). This is an **all-to-all** communication phase.
4. **Local sort**: each node receives its bucket and sorts it locally (using `std::sort` or radix sort).

The all-to-all phase is the bottleneck. For n total elements across P nodes, each node sends ~n/P² elements to each other node. Total data transferred: n/P × (P-1) ≈ n per node across the network. This is unavoidable — to globally sort, every element must potentially move to a different node.

Total time: O(sampling + broadcast + (n/P) all-to-all + (n/P) log(n/P)). The all-to-all term dominates for large n. With InfiniBand (100 Gbps): 1 TB / 100 Gbps ≈ 80 seconds minimum for the data transfer alone. Real performance: ~2-3 minutes for 1 TB on 100 nodes.

### Radix Sort (Distributed)

For integer keys, distributed radix sort avoids the sample step:

1. Each pass sorts by one byte (like LSD radix sort).
2. All-to-all communication based on the byte value: elements with byte = k go to node (k % P).
3. After 4 passes (for 32-bit integers), the data is globally sorted.

No sampling overhead, no load imbalance from bad splitters. But requires 4 all-to-all rounds, each shuffling the entire dataset. For 1 TB: 4 × 1 TB transferred = 4 TB. On 100 Gbps: ~320 seconds. Worse than sample sort unless the data is already well-distributed.

### Merge Sort (Distributed)

For data that's already partially sorted (e.g., log files by timestamp):

1. Each node sorts its local data.
2. A merge tree: nodes pair up, exchange data, and merge. The pairs pair up again, recursively.
3. After O(log P) rounds, the root node has the fully sorted array.

Better when data is already partitioned (no all-to-all shuffle). Total data transferred: each element participates in O(log P) merges × n/P elements per merge. Total: O(n log P) — worse than sample sort's O(n) for large P.

## Distributed Matrix Multiplication

Multiplying two n×n matrices across P nodes. The problem size is 2n² (input) + n² (output) = 3n² numbers.

### Cannon's Algorithm (2D Grid)

Assume P is a perfect square: √P × √P grid of processes. Each process initially holds submatrices A_sub and B_sub of size n/√P × n/√P.

1. **Initial shift**: skew A's rows (row i shifts left by i) and B's columns (column j shifts up by j).
2. **For √P iterations**:
   a. Multiply local A_sub × B_sub, accumulate into C.
   b. Shift A left by 1, shift B up by 1.
3. After √P iterations, each process has its block of C.

Communication: √P shifts × 2 matrices × n²/P elements per shift = 2n²/√P total data moved per node. Total across all nodes: O(n²√P). Compare with sequential: O(n³) computation, O(n²) memory.

For large n, computation dominates: O(n³/P) per process, O(√P) rounds of O(n²/P) communication. The ratio of computation to communication: O(n/√P). For n = 10⁵ and P = 100: ratio ≈ 10⁵/10 = 10⁴. Communication is negligible.

### SUMMA (Scalable Universal Matrix Multiply)

More flexible than Cannon (doesn't require a square grid):

1. For each row of blocks in A and corresponding column of blocks in B:
   a. Broadcast A's block across its row of processes.
   b. Broadcast B's block across its column of processes.
   c. Each process multiplies the received blocks and accumulates into C.

Communication: n/b blocks × 2 broadcasts per block × log(√P) × n²/P. With block size b = n/√P: O(n² log P / √P). Similar to Cannon but works on rectangular grids and for non-square matrices.

## Distributed Graph Algorithms

Graph algorithms are hard to distribute because graphs have irregular structure. Two vertices connected by an edge may reside on different nodes, requiring communication.

### Parallel BFS (Distributed)

Same level-synchronous approach as shared-memory BFS, but the frontier is distributed:

```
Each node:
  while frontier not empty:
    for each local vertex u in frontier:
      for each neighbor v of u:
        if v is local and not visited:
          mark v visited, add to next_frontier
        if v is remote (on node N):
          send message to N: "visit v from u"
    barrier (all nodes sync)
    for each received "visit v" message:
      if v not visited:
        mark v visited, add to next_frontier
    frontier = next_frontier
```

The communication pattern: each level of BFS may send messages to any node. For a graph with high-degree vertices (e.g., social networks with celebrity nodes having millions of followers), a single vertex can generate millions of messages — the communication is highly skewed.

Optimizations:
- **Bitmap frontiers**: represent the frontier as a distributed bitmap. The 1D partitioning case: n/P bits per node.
- **Direction optimization**: for large frontiers (many nodes being explored), switch to "pull" mode — each unvisited node checks if any neighbor is in the frontier, rather than having the frontier push to neighbors. This reduces communication when the frontier is large and most edges are already explored.
- **2D partitioning**: partition the adjacency matrix into P_r × P_c blocks. Each edge is assigned to one process. This balances communication better than 1D (vertex) partitioning for power-law graphs.

### PageRank (Distributed)

PageRank is iterative and communication-bound. Each iteration:
1. Each node computes its vertices' PageRank contributions to neighbors.
2. Shuffle: send contributions to the nodes that own those neighbors.
3. Each node sums incoming contributions and computes new PageRank values.

The shuffle step is an all-to-all communication similar to MapReduce shuffle. For a web graph with 1B pages and 10B edges: each iteration sends ~10B floating-point numbers = 40 GB. On 100 nodes with 10 Gbps: ~3.2 seconds per iteration. For 100 iterations: ~5 minutes. This is why Google precomputes PageRank offline.

## Key Lessons

1. **Data distribution determines communication.** If the data is partitioned to minimize cross-node edges, communication is low. Finding a good partition (graph partitioning, hypergraph partitioning) is NP-hard, but heuristics (METIS, ParMETIS) work well in practice.
2. **Sample sort is the practical distributed sort.** One all-to-all shuffle, then local sort. Radix sort does 4× the data transfer. Merge sort does O(log P)×. For general keys, sample sort is optimal.
3. **Matrix algorithms scale well.** The computation O(n³/P) grows faster than communication O(n²/√P). For large n, matrix multiply is effectively computation-bound even across nodes.
4. **Graph algorithms scale poorly.** The computation is O(E/P) but communication can be O(E). Power-law graphs make this worse — a few vertices have most edges, creating communication hotspots.
5. **Distributed algorithms are about minimizing rounds.** Each communication round costs α (latency) + βn (bandwidth). For large α (WAN: 100 ms), reducing rounds from 100 to 10 is a 10× speedup even with the same total bandwidth.
