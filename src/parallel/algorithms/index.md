# Parallel Algorithms

Parallel algorithms are not sequential algorithms with `#pragma omp parallel` slapped on. They require fundamentally different approaches — divide-and-conquer, work-efficient parallel primitives, and an understanding of work and span. A parallel algorithm with span O(n) on 1000 cores is no faster than sequential. This article covers the essential primitives and analysis tools.

## Work and Span (T₁ / T∞)

Every parallel algorithm has:
- **Work** (T₁): total number of operations. Same as the sequential complexity.
- **Span** (T∞): length of the critical path — the longest chain of dependencies. The minimum possible time with infinite processors.

Examples for an array of size n:

| Algorithm | Work T₁ | Span T∞ | Parallelism (T₁/T∞) |
|-----------|---------|---------|---------------------|
| Parallel sum (reduce) | O(n) | O(log n) | O(n / log n) |
| Parallel quicksort | O(n log n) | O(log² n) avg | O(n / log n) |
| Parallel merge sort | O(n log n) | O(log² n) | O(n / log n) |
| Parallel prefix sum | O(n) | O(log n) | O(n / log n) |
| Parallel BFS | O(V + E) | O(diameter) | O((V+E)/diameter) |

A parallel algorithm with T₁ = O(n log n) and T∞ = O(n) has parallelism = O(log n) — barely worth parallelizing beyond a few cores. An algorithm with T₁ = O(n) and T∞ = O(log n) has parallelism = O(n / log n) — scalable to thousands of cores.

## Parallel Reduction (Sum)

Summing an array of n numbers:

```
Sequential:   (((a₀ + a₁) + a₂) + a₃) + ... + aₙ₋₁    T₁ = n-1, T∞ = n-1
Parallel:     (((a₀ + a₁) + (a₂ + a₃)) + ((a₄ + a₅) + (a₆ + a₇))) + ...
              Tree structure: T₁ = n-1, T∞ = ceil(log₂ n)
```

Implementation:

```rust
fn parallel_sum(a: &[i32]) -> i32 {
    if a.len() == 1 { return a[0]; }

    let mid = a.len() / 2;
    let (left_slice, right_slice) = a.split_at(mid);

    let (left_sum, right_sum) = rayon::join(
        || parallel_sum(left_slice),
        || parallel_sum(right_slice),
    );

    left_sum + right_sum
}
```

For n = 10⁶, the tree has ~20 levels. With 8 cores: ~2.5 levels per core (8 = 2³, so 3 levels for 8 subtrees, then each core computes sequentially). Span = O(log n), work = O(n). Parallelism = n / log n — excellent scalability.

## Parallel Prefix Sum (Scan)

Prefix sum: given [a₀, a₁, ..., aₙ₋₁], compute [a₀, a₀+a₁, a₀+a₁+a₂, ..., Σ₀ⁿ⁻¹ aᵢ].

The **Blelloch scan** (two-pass tree algorithm):

1. **Up-sweep (reduce)**: build a reduction tree, storing partial sums at each level.
2. **Down-sweep**: propagate prefix sums down the tree. The root gets 0. Left child: parent's prefix. Right child: parent's prefix + left sibling's reduction. Leaves get the final prefix sums.

```
Example: [1, 2, 3, 4, 5, 6, 7, 8]

Up-sweep (reduce):
Level 0 (leaves): [1,  2,  3,  4,  5,  6,  7,  8]
Level 1:          [3,      7,      11,     15]
Level 2:          [10,             26]
Level 3:          [36]

Down-sweep:
Level 3:          36 (root gets 0)
Level 2:          10 (left=0),          26 (right=10)
Level 1:          3 (left=0), 7 (right=3), 11 (left=10), 15 (right=21)
Level 0:          [0, 1, 3, 6, 10, 15, 21, 28]

Final scan:       [1, 3, 6, 10, 15, 21, 28, 36]
```

Work: O(n). Span: O(log n). Two passes, each O(log n) depth.

```rust
use rayon::prelude::*;

fn parallel_prefix_sum(a: &mut [i32]) {
    let n = a.len();
    let a_ptr = a.as_mut_ptr();

    // Up-sweep
    let mut stride = 1usize;
    while stride < n {
        let indices: Vec<usize> = (stride - 1..n)
            .step_by(2 * stride)
            .filter(|&i| i + stride < n)
            .collect();
        indices.into_par_iter().for_each(|i| unsafe {
            *a_ptr.add(i + stride) += *a_ptr.add(i);
        });
        stride *= 2;
    }

    // Down-sweep
    unsafe { *a_ptr.add(n - 1) = 0; }
    stride = n / 2;
    while stride > 0 {
        let indices: Vec<usize> = (stride - 1..n)
            .step_by(2 * stride)
            .filter(|&i| i + stride < n)
            .collect();
        indices.into_par_iter().for_each(|i| unsafe {
            let temp = *a_ptr.add(i);
            *a_ptr.add(i) = *a_ptr.add(i + stride);
            *a_ptr.add(i + stride) += temp;
        });
        stride /= 2;
    }
}
```

This is work-efficient: T₁ = O(n), same as sequential. The span T∞ = O(log n) makes it extremely parallel.

## Parallel Merge Sort

Sequential merge sort: divide array in half, recursively sort, merge. The merge step is O(n) and sequential — this is the critical path.

Parallel merge sort: divide recursively until subarrays fit in L1 cache (~4096 elements), then sort sequentially. The merge uses a parallel merge algorithm (find the median of the merged array, split both inputs, recurse):

```
Parallel merge (A of size m, B of size n):
  If m + n < threshold: sequential merge.
  Find the median element: the element at position (m+n)/2 in the merged array.
  Binary search in A to find where this median comes from → split A and B.
  Spawn left merge, compute right merge, sync.
```

Span: O(log² n) — O(log n) levels of recursion, each performing O(log n) binary searches. Work: O(n log n) — same as sequential but with higher constant factors from the binary search.

In practice, a simpler approach: parallelize only the recursive calls, do sequential merge at each level. The merge is O(n), but if we have P = O(n / log n) processors, the merge work dominates. The sweet spot: parallel recursion until subarrays are ~L1-sized, then sequential merge sort from there.

## Parallel BFS (Breadth-First Search)

BFS is hard to parallelize because each level depends on the previous one. But within a level, all nodes can be processed in parallel:

```
Parallel BFS(graph, source):
  frontier = {source}
  visited[source] = true
  distance = 0
  while frontier not empty:
    next_frontier = {}
    parallel for each node u in frontier:
      for each neighbor v of u:
        if not visited[v]:
          visited[v] = true
          v.distance = distance + 1
          atomic add v to next_frontier  // Or use per-thread buffers
    frontier = next_frontier
    distance++
```

The bottleneck: appending to `next_frontier` needs synchronization. Options:
- **Per-thread buffers**: each thread collects its neighbors, then merge buffers after the parallel loop. Faster than atomics but uses more memory.
- **Compare-and-swap on visited**: use CAS to claim `visited[v]`. If the CAS succeeds, add to next_frontier. Avoids separate buffer merging.
- **Bitmap frontier**: represent the frontier as a bitmap. Process all nodes, check bitmap for "in frontier," set bitmap for "in next frontier." Good for dense graphs.

## Key Lessons

1. **Work must equal sequential work.** A parallel algorithm with T₁ = O(n log n) for a problem that's O(n) sequentially has a work penalty. It needs P > Ω(log n) processors just to break even with sequential.
2. **Span limits scalability.** If T∞ = O(n), no amount of parallelism helps beyond a few cores. The algorithm must be restructured to reduce the critical path.
3. **Parallel primitives compose.** Reduce, scan, filter, map — the same primitives appear everywhere. Master these four and you can parallelize most algorithms.
4. **The constant factors matter.** A parallel algorithm with T₁ = 2n and T∞ = O(log n) beats a sequential algorithm with n operations only if P > 2. For small P, the work penalty eats the speedup.
5. **Cache efficiency trumps parallelism within a core.** Tiling for cache (from earlier chapters) still applies in parallel. A blocked parallel algorithm is faster than a naive parallel one.
