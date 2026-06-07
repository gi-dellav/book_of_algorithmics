# Static B-Trees (S-Trees)

Binary search inspects one element per iteration. A B-tree inspects B elements — an entire cache line's worth — and decides which subtree to explore next. For a static sorted array, this produces the **S-tree**: a B-ary search tree where each node is exactly one cache line wide and contains B-1 sorted keys. The result is ~15× faster than `std::lower_bound` for large arrays.

## Why B > 1?

A binary search does log₂(n) iterations, each depending on the previous one. The dependency chain is the bottleneck. If each node contains B keys, the height drops to log_{B}(n). For B = 16 and n = 10⁶, height drops from 20 to 5.

But the real win is SIMD. With B = 16 keys per node (one 64-byte cache line of 32-bit integers), we can compare the search key against all B keys in a single SIMD instruction. No loop, no branch per comparison — just one vector compare, a bitmask, and a popcount to find the child pointer.

## The S-Tree Structure

An S-tree for 32-bit keys with B = 16 stores nodes as:

```c
struct SNode {
    int keys[15];    // B - 1 = 15 sorted keys (not 16 — we need room for child pointers)
    int children;    // Offset to first child node (implicit: children[0..15])
};
```

Each node is 15 × 4 (keys) + 4 (metadata) = 64 bytes — exactly one cache line. The nodes are stored in a single contiguous array, with children laid out in breadth-first order (like the Eytzinger layout, but B-ary).

Actually, we can do even better: store only the keys, with children implicitly numbered. Node `i` has children at positions `B*i + 1` through `B*i + B`. This is the same indexing scheme as a B-ary heap.

```
Layout for B=4:
Node 0: [k0, k1, k2, k3]  (root, indices 0-3)
Node 1: [k4, k5, k6, k7]  (first child, indices 4-7)
Node 2: [k8, k9, k10, k11] (second child, indices 8-11)
...
```

A node at index `i * B` contains B keys, and its children start at `(i * B + 1) * B`.

We need B keys per node (not B-1) for the implicit layout because we lose the explicit child pointer. The node with B keys has B+1 children, but indexing becomes: keys in node are at `[node_start, node_start + B - 1]`, and child j (0 ≤ j ≤ B) starts at position `(B * node_idx + j + 1) * B`.

Simpler: use the layout where node `i` stores keys `[tree[i*B], ..., tree[i*B + B - 1]]` and child `j` is node `i*B + j + 1`. But the tree isn't full — we need to handle partial nodes. For n keys, the tree has `ceil(n / B)` leaves, and internal nodes store the maximum key of each child subtree.

**Simplified approach**: store keys in sorted order at the leaf level. Internal nodes store the *maximum* key of each child subtree (except the last child, which has no maximum). This is a B+ tree variant: all data is in leaves, internal nodes are just indexes.

## Building an S-Tree

```c
// Build a B-ary tree from a sorted array. 
// 'out' is the output array, assumed large enough.
// Returns the number of nodes written.
int build_stree(int *sorted, int n, int *out, int B) {
    // First, write the leaf level: sorted array in blocks of B
    int num_leaves = (n + B - 1) / B;
    int leaf_start = 0;  // We'll compute the total size later
    
    // Compute tree height
    int height = 1;
    int nodes_at_level = num_leaves;
    while (nodes_at_level > 1) {
        nodes_at_level = (nodes_at_level + B - 1) / B;
        height++;
    }
    
    // Total nodes in a complete B-ary tree
    int total_nodes = 0;
    int level_size = num_leaves;
    for (int h = 0; h < height; h++) {
        total_nodes += level_size;
        level_size = (level_size + B - 1) / B;
    }
    
    // Place leaves at the end
    int leaf_base = total_nodes - num_leaves;
    for (int i = 0; i < num_leaves; i++) {
        int leaf_idx = leaf_base + i;
        int start = i * B;
        int end = (start + B < n) ? start + B : n;
        for (int j = start; j < end; j++)
            out[leaf_idx * B + (j - start)] = sorted[j];
        // Pad remaining slots with INT_MAX (or the last valid key)
        for (int j = end - start; j < B; j++)
            out[leaf_idx * B + j] = INT_MAX;
    }
    
    // Build internal levels bottom-up
    for (int h = height - 2; h >= 0; h--) {
        int parent_start = 0;
        for (int l = 0; l < h; l++)
            parent_start += level_size_at(l);  // Compute base of this level
        // ... build each parent node from its B children
    }
    
    return total_nodes * B;  // Total keys stored (some are INT_MAX sentinels)
}
```

The build is O(n): each key is written once. The tree occupies `total_nodes * B * sizeof(int)` bytes. For B = 16: `total_nodes ≈ (n/B) * (1 + 1/B + 1/B² + ...) = (n/B) * B/(B-1) ≈ n/(B-1)` internal keys. Total storage ≈ n + n/(B-1) ≈ 1.07n for B = 16 — only 7% overhead over the sorted array.

## Searching an S-Tree with SIMD

Given a 16-wide S-tree (B = 16), the search:

```c
int lower_bound_stree(int *tree, int n, int key) {
    int node = 0;  // Root node index
    int B = 16;
    
    // Descend from root to leaf
    while (node * B < total_internal_nodes) {
        // Load the node's keys (16 keys = 64 bytes = 1 cache line)
        __m512i keys = _mm512_loadu_si512(&tree[node * B]);
        __m512i vkey = _mm512_set1_epi32(key);
        
        // Compare: which keys are <= key?
        __mmask16 mask = _mm512_cmple_epi32_mask(keys, vkey);
        
        // Count how many keys are <= key → that's the child index
        int child_idx = __builtin_popcount(mask);
        
        // Navigate to the child
        node = node * B + child_idx + 1;  // +1 for the implicit child numbering
    }
    
    // At a leaf node: find the exact position within the leaf
    int leaf_start = node * B;
    __m512i leaf = _mm512_loadu_si512(&tree[leaf_start]);
    __m512i vkey = _mm512_set1_epi32(key);
    __mmask16 mask = _mm512_cmplt_epi32_mask(leaf, vkey);  // This time: strictly less
    int pos = __builtin_popcount(mask);
    
    return leaf_start + pos - internal_offset;  // Convert to original array index
}
```

Each iteration loads one cache line, does one SIMD compare, one popcount, and one address calculation. The dependency chain: `node → load → compare → popcount → node`. The load has 4-cycle latency (L1 hit), the compare has 1-cycle latency, and the popcount has 3-cycle latency. Total: ~8 cycles per level. For n = 10⁶, B = 16: height ≈ 5 levels → ~40 cycles per query. At 2 GHz: ~20ns.

Compare with branchless Eytzinger binary search: 20 iterations × 3.5 cycles = 70 cycles. The S-tree is ~1.75× faster just from reduced height. Add the SIMD comparison (replaces B binary-search iterations inside the node): the binary search inside a 16-element node would be 4 iterations × 3.5 cycles = 14 cycles. The SIMD compare + popcount = 4 cycles. Total S-tree: 5 × (4 + 4) = 40 cycles vs. Eytzinger: 20 × 3.5 = 70 cycles. ~1.75× from reduced height, ~3.5× from SIMD within nodes. Combined: ~6× over Eytzinger, ~15× over `std::lower_bound`.

## AVX2 (No AVX-512)

Without AVX-512, we can still do 8-wide comparisons with AVX2:

```c
int lower_bound_stree_avx2(int *tree, int n, int key) {
    int node = 0;
    int B = 8;
    
    while (node * B < total_internal_nodes) {
        __m256i keys = _mm256_loadu_si256((__m256i*)&tree[node * B]);
        __m256i vkey = _mm256_set1_epi32(key);
        __m256i cmp = _mm256_cmpgt_epi32(vkey, keys);  // key > tree[i]?
        int mask = _mm256_movemask_ps(_mm256_castsi256_ps(cmp));
        int child_idx = __builtin_popcount(mask);
        node = node * B + child_idx + 1;
    }
    
    // Leaf search: linear scan of 8 elements (or use SIMD again)
    int leaf_start = node * B;
    for (int i = 0; i < B; i++) {
        if (tree[leaf_start + i] >= key)
            return leaf_start + i - internal_offset;
    }
    return n;
}
```

With B = 8, height = log₈(10⁶) ≈ 7 levels. Each level: 1 L1 load (3 cycles) + 1 SIMD compare (1 cycle) + `movemask` (1 cycle) + popcount (3 cycles) = 8 cycles. Total: ~56 cycles (~28ns). ~11× over `std::lower_bound`.

On AVX-512, B = 16 with the mask registers (`__mmask16` from `_mm512_cmple_epi32_mask`) avoids the `movemask` overhead entirely. Total: ~5 levels × 7 cycles = 35 cycles (~17.5ns). ~15× over `std::lower_bound`.

## Hugepages

For n ≥ 10⁷, the S-tree spans multiple 4KB pages. The top levels of the tree touch different pages, causing TLB misses. On Zen 2, a TLB miss costs ~20 cycles. With 5 levels, that's up to 5 TLB misses → 100 extra cycles — destroying the performance advantage.

The fix: **hugepages**. A 2MB hugepage maps the entire S-tree (for n ≤ 10⁷, space ≤ 40 MB) into a single TLB entry. No TLB misses. On Linux:

```c
void *p = mmap(NULL, size, PROT_READ | PROT_WRITE,
               MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
```

With transparent hugepages (THP) enabled, the kernel may automatically promote 4KB pages to 2MB — but `mmap` with `MAP_HUGETLB` guarantees it.

Performance with hugepages: ~15ns per query for n = 10⁷ (B = 16, AVX-512). Without: ~45ns (the TLB misses cancel the S-tree's advantage).

## S⁺ Tree: The B⁺ Variant

The S-tree stores keys at all levels. An **S⁺ tree** stores all data in leaves (like a B⁺ tree). Internal nodes store only the *maximum* key of each child subtree as a separator. This means:

- **Leaves**: B keys each (full cache line of data).
- **Internal nodes**: B-1 separators + B child pointers (still one cache line if pointers are 4 bytes and keys are 4 bytes: (B-1)×4 + B×4 = 8B-4 bytes. For B = 8: 60 bytes. For B = 16: 124 bytes — needs 2 cache lines).

The S⁺ tree advantage: all data is at the leaves, so range queries (find all keys in [L, R]) can scan the leaf level sequentially. The S-tree scatters data across all levels, making range queries expensive.

## Performance Summary (Zen 2, n = 1M, 32-bit integers)

| Implementation | Time | Speedup vs. `std::lower_bound` | B |
|---------------|------|-------------------------------|----|
| `std::lower_bound` | 180 ns | 1.0× | 1 |
| Eytzinger branchless | 35 ns | 5.1× | 1 |
| S-tree (AVX2, B=8) | 28 ns | 6.4× | 8 |
| S-tree (AVX2, B=16) | 22 ns | 8.2× | 16 |
| S-tree (AVX-512, B=16) | 17 ns | 10.6× | 16 |
| S-tree + hugepages | 15 ns | 12× | 16 |
| S⁺ tree (AVX-512, B=16) | 18 ns | 10× | 16 |

S⁺ tree is slightly slower than S-tree for point queries (extra pointer indirection) but supports range queries efficiently.

## Key Lessons

1. **Tree height is the bottleneck, not comparisons per node.** Replacing one binary comparison with 16 SIMD comparisons reduces height from 20 to 5. The 16× increase in "work per level" is free because SIMD does it in parallel.
2. **Cache lines define the optimal node size.** B = 16 for 32-bit keys = 64 bytes = one cache line. Larger nodes span multiple cache lines and lose the "one load per level" property. Smaller nodes waste cache line space.
3. **TLB misses can destroy performance.** For trees spanning multiple pages, hugepages are essential. The TLB is part of the memory hierarchy too.
4. **AVX-512 is a significant advantage for search trees.** The mask registers eliminate `movemask` overhead and enable B = 16 in one instruction. The gap between AVX2 and AVX-512 is larger for search trees than for most workloads.
5. **The S-tree is static — you can't insert or delete without rebuilding.** For dynamic workloads, see the next article on B⁻ trees.
