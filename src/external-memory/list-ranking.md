# List Ranking

List ranking is the problem of computing, for each element in a linked list, its distance from the head (or tail). It sounds trivial — just walk the list — but it's surprisingly difficult in the external memory model because following pointers causes random I/O.

## The Problem

Given a linked list of N nodes, each with a pointer `next` to the successor (and a sentinel with `next = NULL`), compute the rank of each node (0 for head, 1 for next, ..., N−1 for tail).

Naive solution: start at head, follow `next` pointers, counting. I/O cost: Θ(N) random accesses — each node may be on a different disk block.

## The Algorithm: Independent Set Removal

Key insight: if we can shortcut over some nodes, we reduce the list size. Then recursively rank the smaller list, and fill in the removed nodes' ranks from their neighbors.

**Step 1: Find an independent set.** A set I of nodes such that no two are adjacent (no node in I points to another node in I). A random independent set can be found by flipping a coin for each node. On average, 1/4 of nodes are in I (they got heads and their predecessor got tails).

**Step 2: Remove independent set.** For each node v in I with predecessor u and successor w:
- `rank[v] = rank[u] + 1`
- `next[u] = w` (shortcut v)

**Step 3: Recursively rank the reduced list** (size ≈ 3N/4).

**Step 4: Restore ranks.** For each removed node v: `rank[v] = rank[u] + 1` (where u is v's original predecessor, now pointing to v's original successor).

After O(log N) recursive steps, the list is reduced to a single node. Total I/O: O((N/B) log_{M/B}(N/B)) — sorting bound. (The independent set selection and shortcutting require sorting the nodes by index to bring adjacent list elements into the same block — this is where the sorting cost comes from.)

## Why This Matters

List ranking is the basis for:
- **Euler tour traversal** of trees: Convert a tree to a linked list via an Euler tour (each node appears multiple times, once per incident edge). Rank the list → compute depth, subtree size, pre/post order for every node.
- **Parallel tree computations**: List ranking is highly parallel (independent set removal can be done in parallel for many nodes). This is used in GPU and parallel algorithms.
- **Functional graph analysis**: Finding cycles, computing distances in directed graphs with out-degree 1.

## The Euler Tour Technique

Given a tree, create a list where each edge (u,v) appears twice: once as `(u,v)` in the direction u→v, once as `(v,u)` in the direction v→u. The Euler tour of the tree visits each edge twice. If we traverse the tour, we visit nodes in DFS order.

To compute subtree sizes:
1. Build the Euler tour list (2E nodes for E edges).
2. Assign weight +1 to forward edges (entering a subtree) and weight −1 to backward edges (leaving a subtree).
3. Compute prefix sums along the list (list ranking with weights).
4. Subtree size of v = prefix sum at exit of v − prefix sum at entry of v + 1.

Without list ranking, this is O(N log N) in external memory (using BFS/DFS with random I/O). With list ranking, it's O((N/B) log_{M/B}(N/B)) — sorting bound. For N = 10⁹ on SSD, the difference is hours vs. minutes.

## Practical Considerations

List ranking is primarily a theoretical tool for designing external-memory algorithms for graph problems. The constant factors in the I/O-efficient version are large (multiple sorting passes). For in-memory computation, the naive pointer-chasing is faster than the I/O-efficient version because random access in RAM is still reasonably fast (100 ns).

But when the list truly doesn't fit in RAM (billions of nodes, stored on SSD), the I/O-efficient algorithm is the only viable approach. The transition happens when N > M × log(N), roughly — when the working set exceeds memory by enough that the random I/Os dominate.
