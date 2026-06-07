# Segment Trees

Segment trees answer range queries — sum, minimum, maximum, greatest common divisor — over any contiguous subarray, with point updates, in O(log n) time. They're the workhorse of competitive programming, but the textbook implementation (pointer-based recursion) is catastrophically slow on real hardware. This article traces the evolution from pointer-based to implicit to Fenwick tree to wide B-ary SIMD segment trees, with speedups exceeding 200× on prefix sum queries.

## The Problem

Given an array `a[0..n-1]` and an associative operation ⊕ (addition, min, max, etc.), support:

- **Query(l, r)**: compute a[l] ⊕ a[l+1] ⊕ ... ⊕ a[r-1].
- **Update(i, x)**: set a[i] = x.

A naive array answers Query in O(n) and Update in O(1). A prefix sum array answers Query in O(1) but Update in O(n). A segment tree answers both in O(log n).

## Stage 1: Pointer-Based Recursive Segment Tree

```c
struct SegNode {
    int lo, hi;          // Range this node covers
    int val;             // Aggregate over [lo, hi)
    SegNode *left, *right;
};

SegNode *build(int *a, int lo, int hi) {
    SegNode *node = malloc(sizeof(SegNode));
    node->lo = lo; node->hi = hi;
    if (hi - lo == 1) {
        node->val = a[lo];
        node->left = node->right = NULL;
    } else {
        int mid = lo + (hi - lo) / 2;
        node->left = build(a, lo, mid);
        node->right = build(a, mid, hi);
        node->val = node->left->val + node->right->val;
    }
    return node;
}

int query(SegNode *node, int ql, int qr) {
    if (ql <= node->lo && node->hi <= qr)  // Fully covered
        return node->val;
    if (qr <= node->lo || node->hi <= ql)  // No overlap
        return 0;  // Identity for sum
    return query(node->left, ql, qr) + query(node->right, ql, qr);
}

void update(SegNode *node, int idx, int val) {
    if (node->hi - node->lo == 1) {
        node->val = val;
    } else {
        int mid = node->lo + (node->hi - node->lo) / 2;
        if (idx < mid) update(node->left, idx, val);
        else           update(node->right, idx, val);
        node->val = node->left->val + node->right->val;
    }
}
```

Performance (Zen 2, n = 100K, random sum queries on [0, n)): ~850ns per query. The bottlenecks:

- **Recursion**: function call overhead (~20 cycles per call), ~2 log n calls per query = ~40 calls = ~800 cycles.
- **Pointer chasing**: each node is separately allocated → nodes are scattered in memory → every access is a potential cache miss.
- **Branch mispredictions**: the overlap check (`ql <= node->lo && node->hi <= qr`) branches unpredictably.
- **Memory**: 6 fields × 8 bytes = 48 bytes per node × 2n nodes = 96n bytes. For n = 100K: ~9.6 MB — fits in L3 but not L2.

## Stage 2: Implicit (Array-Based) Segment Tree

Store the tree in a flat array, like a binary heap. Index 1 is the root; children of index `i` are `2i` and `2i+1`.

```c
int tree[4 * MAXN];  // Worst-case size for an implicit segment tree

void build_implicit(int *a, int n) {
    // Copy a[] to the leaves
    for (int i = 0; i < n; i++)
        tree[n + i] = a[i];  // Leaves start at index n
    // Build internal nodes bottom-up
    for (int i = n - 1; i > 0; i--)
        tree[i] = tree[2*i] + tree[2*i+1];
}

int query_implicit(int n, int l, int r) {
    int res = 0;
    for (l += n, r += n; l < r; l >>= 1, r >>= 1) {
        if (l & 1) res += tree[l++];  // l is a right child → include it
        if (r & 1) res += tree[--r];  // r is a right child → include r-1 (left child)
    }
    return res;
}

void update_implicit(int n, int idx, int val) {
    tree[n + idx] = val;
    for (int i = (n + idx) / 2; i > 0; i /= 2)
        tree[i] = tree[2*i] + tree[2*i+1];
}
```

Performance: ~180ns per query (~4.7× faster than pointer-based). Eliminates:
- Recursion (iterative bottom-up).
- Pointer chasing (sequential array, good locality).
- Memory overhead: 4n ints = 16n bytes (1.6 MB for n = 100K — fits in L2).

The remaining overhead: the loop does O(log n) iterations with unpredictable branches (`if (l & 1)`).

## Stage 3: Branchless Bottom-Up

The `if (l & 1)` and `if (r & 1)` branches can be eliminated with conditional moves:

```c
int query_branchless(int n, int l, int r) {
    int res = 0;
    for (l += n, r += n; l < r; l >>= 1, r >>= 1) {
        int add_l = (l & 1) ? tree[l++] : 0;
        int add_r = (r & 1) ? tree[--r] : 0;
        res += add_l + add_r;
    }
    return res;
}
```

The compiler may or may not turn the ternary into `cmov`. On Clang with `-O3`: it does. On GCC: sometimes not. Explicit asm or `__builtin_expect` can help.

Performance: ~120ns per query (~7× faster than pointer-based). The branch misprediction cost is gone. But we still have the loop dependency: `res` depends on the previous iteration's `res`, preventing ILP.

## Stage 4: Fenwick Tree (Binary Indexed Tree)

For **prefix sum** queries (l = 0), the Fenwick tree is even simpler and faster:

```c
int fenwick[MAXN];

void fenwick_add(int n, int idx, int delta) {
    for (idx++; idx <= n; idx += idx & -idx)
        fenwick[idx - 1] += delta;  // 0-indexed variant
}

int fenwick_sum(int n, int idx) {  // Sum of a[0..idx)
    int res = 0;
    for (; idx > 0; idx -= idx & -idx)
        res += fenwick[idx - 1];
    return res;
}

int fenwick_range(int n, int l, int r) {
    return fenwick_sum(n, r) - fenwick_sum(n, l);
}
```

The Fenwick tree uses exactly n elements (not 2n or 4n). Each query touches indices that follow the binary representation: `idx = 13 = 1101₂ → 1100 → 1000 → 0` (three accesses for 13). Average: log₂(n)/2 accesses per prefix query.

Performance: ~35ns per prefix sum (~24× faster than pointer-based). Two prefixes = 70ns for a range query. For updates: ~25ns.

But the Fenwick tree has a subtle cache problem: powers of two are accessed frequently. Index 8 (1000₂) is accessed by every query for idx ≥ 8. Index 4 is accessed by every query for idx ∈ [4,8). The hottest indices cause cache associativity conflicts — multiple hot indices map to the same cache set, causing evictions.

### The Cache Associativity Fix

On Zen 2, the L1 data cache is 32KB, 8-way set associative, 64-byte lines → 64 sets. Indices 0, 64, 128, 192, ... map to set 0. Indices 1, 65, 129, ... map to set 1. The Fenwick tree's hot indices (powers of two: 1, 2, 4, 8, 16, 32, 64, 128, ...) map to different sets, so this isn't a problem. But for large n, indices like 1024, 2048, 4096 all map to set 0 (if offset by zero) and evict each other.

The fix: add "holes" to the Fenwick tree to shift the alignment. Instead of storing `fenwick[idx]` at position `idx`, store at `idx + (idx / CACHE_LINE_SIZE)`. This spreads hot indices across different cache sets. Performance gain: ~10% for n > 10⁵.

## Stage 5: Wide Segment Trees (B-ary)

Why stop at binary? A B-ary segment tree stores B children per node, reducing height from log₂(n) to log_B(n). For B = 16: height drops from ~17 to ~3.

```c
#define B 16

int wide_tree[4 * MAXN];  // Still 1-indexed, but B-ary

void build_wide(int *a, int n) {
    // Pad n to a power of B
    int padded = 1;
    while (padded < n) padded *= B;
    
    // Place leaves
    for (int i = 0; i < n; i++)
        wide_tree[padded + i] = a[i];
    for (int i = n; i < padded; i++)
        wide_tree[padded + i] = 0;  // Identity
    
    // Build bottom-up: each parent is the sum of its B children
    for (int i = padded - 1; i > 0; i--)
        wide_tree[i] = wide_tree[B*i] + wide_tree[B*i+1] + ... + wide_tree[B*i+B-1];
}

int query_wide(int padded, int l, int r) {
    int res = 0;
    l += padded; r += padded;
    
    while (l < r) {
        // Find the largest aligned block covering part of [l, r)
        // This is more complex than binary...
        // For each level, check if l is at a block boundary
        int block_start = l;
        while (l % B != 0 && l < r) {
            res += wide_tree[l++];
        }
        while (r % B != 0 && l < r) {
            res += wide_tree[--r];
        }
        l /= B; r /= B;
    }
    return res;
}
```

The wide segment tree is trickier to query than binary because blocks don't align nicely with arbitrary ranges. The query must process partial blocks at the edges and whole blocks in the middle. But with SIMD, processing a partial block of up to B elements can be done with a single vector load + mask:

```c
int query_wide_simd(int padded, int l, int r) {
    int res = 0;
    l += padded; r += padded;
    
    while (l < r) {
        // Partial block at the left edge
        if (l % B != 0) {
            int block = l / B;
            int offset = l % B;
            int count = (B - offset < r - l) ? B - offset : r - l;
            
            // Load the block, mask off elements before 'offset' and after 'count'
            __m512i v = _mm512_loadu_si512(&wide_tree[block * B]);
            __mmask16 mask = (1 << count) - 1;
            mask <<= offset;
            __m512i masked = _mm512_maskz_mov_epi32(mask, v);
            res += _mm512_reduce_add_epi32(masked);  // Horizontal sum
            
            l += count;
            continue;
        }
        
        // Full block
        if (l + B <= r) {
            __m512i v = _mm512_loadu_si512(&wide_tree[l]);
            res += _mm512_reduce_add_epi32(v);
            l += B;
            continue;
        }
        
        // Partial block at the right edge
        if (r % B != 0 && l < r) {
            int block = r / B;
            int count = r % B;
            __m512i v = _mm512_loadu_si512(&wide_tree[block * B]);
            __mmask16 mask = (1 << count) - 1;
            __m512i masked = _mm512_maskz_mov_epi32(mask, v);
            res += _mm512_reduce_add_epi32(masked);
            break;
        }
        
        l /= B; r /= B;
    }
    return res;
}
```

Performance: ~15ns per query for n = 100K, B = 16. ~57× faster than pointer-based.

## Stage 6: Non-Reversible Monoids and Lazy Propagation

Sum is a **reversible** operation: you can compute range sum as `prefix[r] - prefix[l]`. But `min` and `gcd` are not reversible — there's no inverse operation. For these, the segment tree must explicitly compute the range aggregate.

Also, range updates (add X to all elements in [l, r)) require **lazy propagation**: mark a node as "dirty" with a pending update, and only propagate it when the node is queried:

```c
int tree[4*MAXN], lazy[4*MAXN];

void apply(int idx, int val, int len) {
    tree[idx] += val * len;  // For sum; for min: tree[idx] += val
    lazy[idx] += val;
}

void push(int idx, int lo, int hi) {
    if (lazy[idx] != 0) {
        int mid = lo + (hi - lo) / 2;
        apply(2*idx, lazy[idx], mid - lo);
        apply(2*idx+1, lazy[idx], hi - mid);
        lazy[idx] = 0;
    }
}

void range_add(int idx, int lo, int hi, int ql, int qr, int val) {
    if (ql <= lo && hi <= qr) {
        apply(idx, val, hi - lo);
        return;
    }
    if (qr <= lo || hi <= ql) return;
    push(idx, lo, hi);
    int mid = lo + (hi - lo) / 2;
    range_add(2*idx, lo, mid, ql, qr, val);
    range_add(2*idx+1, mid, hi, ql, qr, val);
    tree[idx] = tree[2*idx] + tree[2*idx+1];
}
```

Lazy propagation doubles the memory (two arrays) and adds a push step to queries. But for range updates, it's essential — without it, range add is O(n log n). With lazy propagation: O(log n) per range update.

## Performance Summary (Zen 2, n = 100K, random range sum queries)

| Implementation | Query Time | Update Time | Speedup (Query) |
|---------------|-----------|-------------|-----------------|
| Pointer-based recursive | 850 ns | 820 ns | 1× |
| Implicit array | 180 ns | 170 ns | 4.7× |
| Branchless implicit | 120 ns | 140 ns | 7× |
| Fenwick tree (prefix) | 35 ns | 25 ns | 24× (prefix) |
| Wide B-ary (B=16, SIMD) | 15 ns | 65 ns | 57× |
| Wide B-ary (B=8, AVX2) | 22 ns | 50 ns | 39× |

For range sum where Fenwick is applicable: use Fenwick. It's smaller, faster, and simpler. For non-reversible operations (min, max, gcd): use the implicit or wide segment tree. For range updates: use lazy propagation with the implicit tree.

## Key Lessons

1. **Recursion is expensive for high-throughput queries.** The call/ret overhead and unpredictable branches in the recursive segment tree cost ~800ns per query. The iterative version is 5× faster just by removing recursion.
2. **Memory layout is everything.** The pointer-based tree scatters 48-byte nodes across the heap — cache misses on every access. The implicit array packs everything into a contiguous array — L2 hits for most queries.
3. **Branchless pays for small-loop data structures.** The `if (l & 1)` branch is 50% predictable in the inner loop. Eliminating it with `cmov` gives ~1.5× improvement.
4. **Fenwick trees are optimal for prefix sums.** They use the minimum memory (n elements), have the shortest inner loop (3 instructions), and benefit from the "power of two" access pattern that's cache-friendly for small n. The associativity fix is a beautiful application of the cache architecture chapter.
5. **SIMD acceleration works for non-trivial data structures.** Loading a full B-ary node with a single SIMD load and using masked operations to handle partial blocks brings query times down to ~15ns — competitive with raw array access.
6. **The operation's properties determine the data structure.** Reversible operations → Fenwick or prefix sum. Non-reversible but associative → segment tree. Range updates → lazy propagation. Know your monoid.
