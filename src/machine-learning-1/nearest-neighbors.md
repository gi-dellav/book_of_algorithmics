# Nearest Neighbors and Clustering

When you don't want to train a model at all, use nearest neighbors. Store the training data, and at prediction time, find the k closest points. The challenge is making "find the k closest" fast — naive O(n) search is unacceptable for n > 10⁵. This article covers exact and approximate nearest neighbor search, plus clustering algorithms that leverage the same spatial indexing structures.

## Exact Nearest Neighbors: k-d Trees

A k-d tree recursively splits the data along alternating dimensions at the median, creating a binary search tree over spatial regions:

```rust
struct KDTree {
    point: Vec<f64>,
    left: Option<Box<KDTree>>,
    right: Option<Box<KDTree>>,
    split_dim: usize,
}

fn build_kdtree(points: &mut [Vec<f64>], depth: usize) -> Option<Box<KDTree>> {
    if points.is_empty() { return None; }

    let dim = depth % points[0].len();
    points.sort_by(|a, b| a[dim].partial_cmp(&b[dim]).unwrap());
    let median = points.len() / 2;

    let point = points[median].clone();
    let left = build_kdtree(&mut points[..median].to_vec(), depth + 1);
    let right = build_kdtree(&mut points[median + 1..].to_vec(), depth + 1);

    Some(Box::new(KDTree { point, left, right, split_dim: dim }))
}

fn knn_search(tree: &KDTree, query: &[f64], k: usize) -> Vec<(f64, usize)> {
    let mut best = BinaryHeap::new(); // max-heap of (-dist, index)

    // Recursive search with pruning
    search_recursive(tree, query, k, &mut best);
    best.into_sorted_vec()
}

fn search_recursive(node: &KDTree, query: &[f64], k: usize,
                    best: &mut BinaryHeap<(OrderedFloat<f64>, usize)>) {
    let dist = euclidean(query, &node.point);
    best.push((OrderedFloat(dist), /* index */));
    if best.len() > k { best.pop(); }

    let dim = node.split_dim;
    let diff = query[dim] - node.point[dim];

    // Search the closer child first
    let (near, far) = if diff < 0.0 {
        (&node.left, &node.right)
    } else {
        (&node.right, &node.left)
    };

    if let Some(near_child) = near {
        search_recursive(near_child, query, k, best);
    }

    // Only search far child if it could contain a closer point
    if let Some(far_child) = far {
        let worst_dist = best.peek().map(|(d, _)| d.0).unwrap_or(f64::INFINITY);
        if diff.abs() < worst_dist {
            search_recursive(far_child, query, k, best);
        }
    }
}
```

k-d trees give O(log n) queries for low-dimensional data (d < 10). For d > 20, the "curse of dimensionality" makes them degrade to O(n) — almost every query explores most of the tree.

## Approximate Nearest Neighbors: HNSW

Hierarchical Navigable Small World (HNSW) is the state-of-the-art for approximate nearest neighbor search. It builds a multi-layer graph where each layer is a navigable small-world graph (like a social network: most nodes connect locally, a few connect across the space):

```rust
struct HNSW {
    layers: Vec<Vec<Vec<usize>>>, // adjacency lists per layer per node
    points: Vec<Vec<f32>>,
    entry_point: usize,
    max_layer: usize,
    ef_construction: usize, // beam width during construction
    ef_search: usize,       // beam width during search
    m: usize,               // max connections per node per layer
}
```

Search: start at the top-layer entry point, greedily descend to the nearest neighbor in each layer, then do beam search in the bottom layer:

```rust
fn hnsw_search(index: &HNSW, query: &[f32], k: usize) -> Vec<(f32, usize)> {
    let mut ep = index.entry_point;

    // Descend through layers
    for lc in (1..=index.max_layer).rev() {
        ep = greedy_search_layer(index, query, ep, lc, 1)[0];
    }

    // Beam search in the bottom layer (layer 0)
    beam_search_layer(index, query, ep, 0, k, index.ef_search)
}
```

HNSW achieves >95% recall with <0.1 ms per query on 10⁶ 128-dimensional vectors. This is what powers vector search in Weaviate, Qdrant, and most vector databases. The construction is O(n log n), query is O(log n) in practice even for d = 100–1000.

## Product Quantization (IVF-PQ)

For billion-scale vector search, HNSW's memory footprint (graph + vectors) becomes prohibitive. Product Quantization compresses each vector into a short code:

1. Split each d-dimensional vector into M sub-vectors of dimension d/M.
2. Run k-means on each sub-vector space independently (typically 256 clusters each).
3. Encode each vector as M bytes (one cluster index per sub-vector).

Distance computation: precompute a lookup table of distances between the query's sub-vectors and all 256 centroids, then sum across M sub-vectors. This is 6–12× faster than computing the full distance (M lookups + M additions vs. d multiplications + d additions).

```rust
struct IVF_PQ {
    coarse_centroids: Vec<Vec<f32>>, // ~√n inverted file centroids
    inverted_lists: Vec<Vec<u8>>,    // PQ codes per cell
    pq_codebook: Vec<Vec<Vec<f32>>>, // [M][256][d/M] centroid table
}

fn ivf_pq_search(index: &IVF_PQ, query: &[f32], n_probe: usize, k: usize) -> Vec<(f32, usize)> {
    // Find n_probe nearest coarse centroids
    let nearest_cells = find_nearest_centroids(query, &index.coarse_centroids, n_probe);

    // Precompute PQ distance table: distances[j][c] = dist(query_sub[j], centroid_j[c])
    let dist_table = precompute_pq_table(query, &index.pq_codebook);

    let mut best = BinaryHeap::new();
    for cell_id in nearest_cells {
        for (vec_id, code) in index.inverted_lists[cell_id].iter().enumerate() {
            // Approximate distance via PQ
            let dist = pq_distance(code, &dist_table);
            best.push((OrderedFloat(dist), vec_id));
            if best.len() > k { best.pop(); }
        }
    }
    best.into_sorted_vec()
}
```

IVF-PQ compresses 128-dimensional float vectors from 512 bytes to ~16 bytes (M=16, 1 byte per sub-vector). A billion vectors fit in 16 GB — a single machine. Query time with n_probe=64: ~5 ms. Recall: ~95% at these settings.

## k-Means Clustering

k-means alternates between assigning points to the nearest centroid and recomputing centroids as the mean of their assigned points:

```rust
fn kmeans(points: &[Vec<f32>], k: usize, max_iter: usize) -> Vec<Vec<f32>> {
    let n = points.len();
    let d = points[0].len();

    // Initialize centroids (k-means++)
    let mut centroids = kmeans_plusplus_init(points, k);

    let mut assignments = vec![0usize; n];
    let mut centroid_sums = vec![vec![0.0f32; d]; k];
    let mut centroid_counts = vec![0usize; k];

    for _ in 0..max_iter {
        // Assignment step: find nearest centroid for each point
        // This is O(nkd) — the bottleneck
        for i in 0..n {
            let mut best_dist = f32::INFINITY;
            let mut best_c = 0usize;
            for c in 0..k {
                let dist = squared_euclidean(&points[i], &centroids[c]);
                if dist < best_dist {
                    best_dist = dist;
                    best_c = c;
                }
            }
            assignments[i] = best_c;
        }

        // Update step: recompute centroids
        for c in 0..k {
            centroid_sums[c].fill(0.0);
            centroid_counts[c] = 0;
        }
        for i in 0..n {
            let c = assignments[i];
            for j in 0..d {
                centroid_sums[c][j] += points[i][j];
            }
            centroid_counts[c] += 1;
        }
        for c in 0..k {
            if centroid_counts[c] > 0 {
                for j in 0..d {
                    centroids[c][j] = centroid_sums[c][j] / centroid_counts[c] as f32;
                }
            }
        }
    }
    centroids
}
```

The assignment step is O(nkd) and dominates runtime. SIMD distance computation (8 floats at a time with AVX2) gives ~6× speedup. For n = 10⁶, k = 100, d = 128: 1.28 × 10¹⁰ distance computations per iteration. SIMD: ~5 seconds per iteration. GPU: ~50 ms per iteration.

## DBSCAN: Density-Based Clustering

DBSCAN groups points that are densely connected via ϵ-neighborhoods. Unlike k-means, it discovers the number of clusters automatically and handles arbitrary shapes. The bottleneck is finding all neighbors within radius ϵ:

```rust
fn dbscan(points: &[Vec<f32>], eps: f32, min_pts: usize) -> Vec<Option<usize>> {
    let n = points.len();

    // Build a spatial index (k-d tree, ball tree, or grid)
    let index = BallTree::build(points);

    let mut labels = vec![None; n];
    let mut cluster_id = 0usize;

    for i in 0..n {
        if labels[i].is_some() { continue; }

        let neighbors = index.range_query(&points[i], eps);
        if neighbors.len() < min_pts {
            labels[i] = None; // noise
            continue;
        }

        // Expand cluster
        labels[i] = Some(cluster_id);
        let mut seeds: Vec<usize> = neighbors.iter().map(|&(_, idx)| idx).collect();
        let mut j = 0;
        while j < seeds.len() {
            let q = seeds[j]; j += 1;
            if labels[q] == Some(0xFFFF) { labels[q] = Some(cluster_id); } // was noise
            if labels[q].is_some() { continue; }
            labels[q] = Some(cluster_id);

            let q_neighbors = index.range_query(&points[q], eps);
            if q_neighbors.len() >= min_pts {
                for &(_, idx) in &q_neighbors {
                    if labels[idx].is_none() || labels[idx] == Some(0xFFFF) {
                        seeds.push(idx);
                        if labels[idx].is_none() { labels[idx] = Some(0xFFFF); }
                    }
                }
            }
        }
        cluster_id += 1;
    }
    labels
}
```

With a ball tree or k-d tree, each range query is O(log n) in low dimensions. For n = 10⁶, a full DBSCAN without a spatial index takes O(n²) — hours. With a ball tree: O(n log n) — seconds. The lesson: spatial indexing is the difference between a clustering algorithm working and failing catastrophically.

## When to Use What

| Task | Algorithm | Query time | Memory |
|------|-----------|-----------|--------|
| Exact NN, d < 10, n < 10⁶ | k-d tree | O(log n) | O(nd) |
| Exact NN, d < 100, n < 10⁶ | Ball tree | O(log n) empirical | O(nd) |
| Approximate NN, d < 1000 | HNSW | ~0.1 ms | O(nd + n·M·connections) |
| Approximate NN, d < 1000, n > 10⁸ | IVF-PQ | ~5 ms | O(n·code_bytes) |
| Clustering, k known | k-means (SIMD/GPU) | — | O(nkd) per iter |
| Clustering, k unknown | DBSCAN + ball tree | — | O(n log n) |
| Clustering, hierarchical | Agglomerative + FLANN | — | O(n²) → O(n log n) with index |
