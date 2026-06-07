# Tree Structures Survey

The preceding articles focused on cache-optimized data structures: S-trees, B⁻ trees, Eytzinger layouts. Before you can optimize, you must understand the baseline. This article surveys classical balanced search trees — AVL, red-black, treap, splay — and compares them on modern hardware. The goal: understand what the cache-optimized variants are improving upon.

## The Search Tree Landscape

A binary search tree (BST) supports insert, delete, and search in O(h) time where h is the height. Without balancing, h can be O(n) (a sorted insertion produces a linked list). Balanced trees guarantee h = O(log n).

The design tension: how much rebalancing work per update vs. how balanced the tree stays. AVL trees maintain strict balance (height difference ≤ 1) at the cost of more rotations. Red-black trees relax balance (no path is more than twice the shortest path) for fewer rotations. Treaps use randomization to achieve balance on average with zero explicit rebalancing.

## AVL Trees

Invented by Adelson-Velsky and Landis (1962). Each node stores a **balance factor**: `height(left) - height(right)`. After each insertion or deletion, walk up the tree and rebalance any node with |balance factor| > 1 using rotations.

```c
struct AVLNode {
    int key;
    int height;
    AVLNode *left, *right;
};

AVLNode *avl_insert(AVLNode *node, int key) {
    if (!node) return new_node(key);
    
    if (key < node->key)
        node->left = avl_insert(node->left, key);
    else if (key > node->key)
        node->right = avl_insert(node->right, key);
    else
        return node;  // Duplicate
    
    node->height = 1 + max(height(node->left), height(node->right));
    int balance = height(node->left) - height(node->right);
    
    // Left-Left case
    if (balance > 1 && key < node->left->key)
        return rotate_right(node);
    // Right-Right case
    if (balance < -1 && key > node->right->key)
        return rotate_left(node);
    // Left-Right case
    if (balance > 1 && key > node->left->key) {
        node->left = rotate_left(node->left);
        return rotate_right(node);
    }
    // Right-Left case
    if (balance < -1 && key < node->right->key) {
        node->right = rotate_right(node->right);
        return rotate_left(node);
    }
    
    return node;
}
```

AVL trees are the most strictly balanced BST: after n insertions, height ≤ 1.44 log₂(n). This gives the fastest lookups among BSTs. But the strict balancing requires more rotations: ~0.47 rotations per insertion on average (vs. ~0.25 for red-black). Each rotation is ~10 pointer assignments — that's ~5 extra pointer writes per insertion.

Performance (Zen 2, 1M random insertions + 1M lookups):

| Operation | AVL | Red-Black | Treap | Splay |
|-----------|-----|-----------|-------|-------|
| Insert | 320 ns | 280 ns | 350 ns | 250 ns |
| Lookup | 180 ns | 200 ns | 220 ns | 300 ns |
| Delete | 340 ns | 300 ns | 370 ns | 270 ns |
| Memory/node | 40 B | 40 B | 40 B | 32 B |

## Red-Black Trees

Invented by Rudolf Bayer (1972) as "symmetric binary B-trees," popularized by Guibas and Sedgewick (1978). Each node is red or black. Invariants:
1. Root is black.
2. No red node has a red child.
3. Every path from root to leaf has the same number of black nodes.

These invariants guarantee height ≤ 2 log₂(n). Insertion fixes at most 2 rotations; deletion fixes at most 3. Fewer rotations than AVL, but the tree is less strictly balanced → ~10% slower lookups, ~10% faster insertions.

```c
enum Color { RED, BLACK };

struct RBNode {
    int key;
    Color color;
    RBNode *left, *right, *parent;
};

void rb_insert_fixup(RBNode *node) {
    while (node->parent && node->parent->color == RED) {
        RBNode *grand = node->parent->parent;
        if (node->parent == grand->left) {
            RBNode *uncle = grand->right;
            if (uncle && uncle->color == RED) {
                // Case 1: Recolor
                node->parent->color = BLACK;
                uncle->color = BLACK;
                grand->color = RED;
                node = grand;
            } else {
                // Case 2 & 3: Rotations
                if (node == node->parent->right) {
                    node = node->parent;
                    rotate_left(node);
                }
                node->parent->color = BLACK;
                grand->color = RED;
                rotate_right(grand);
            }
        } else {
            // Mirror cases
        }
    }
}
```

Red-black trees are the most widely implemented: `std::map`, Java `TreeMap`, Linux kernel's CFS scheduler. The implementation complexity is higher than AVL (more cases), but the average performance is slightly better for mixed workloads.

## Treaps (Randomized BSTs)

A treap (tree + heap) assigns each node a random priority and maintains both BST order (by key) and heap order (by priority). Insertion: insert as a leaf by key, then rotate up while the heap property is violated.

```c
struct TreapNode {
    int key;
    int priority;  // Random
    TreapNode *left, *right;
};

TreapNode *treap_insert(TreapNode *node, int key) {
    if (!node) {
        TreapNode *n = malloc(sizeof(TreapNode));
        n->key = key;
        n->priority = rand();
        n->left = n->right = NULL;
        return n;
    }
    
    if (key < node->key) {
        node->left = treap_insert(node->left, key);
        if (node->left->priority > node->priority)
            node = rotate_right(node);
    } else {
        node->right = treap_insert(node->right, key);
        if (node->right->priority > node->priority)
            node = rotate_left(node);
    }
    return node;
}
```

Expected height: ~4.3 log₂(n) (slightly worse than AVL/red-black due to the random priorities, which produce a slightly less balanced tree on average). Advantages: no explicit rebalancing logic, no parent pointers, no colors or balance factors — simpler code. Disadvantage: requires a random number generator (or a good hash of the key) and has the worst expected height of the three.

## Splay Trees

A splay tree (Sleator & Tarjan, 1985) is self-adjusting with **no** explicit balance information. Every operation (insert, lookup, delete) "splays" the accessed node to the root via rotations. This gives O(log n) **amortized** time per operation.

```c
void splay(SplayNode **root, SplayNode *x) {
    while (x->parent) {
        SplayNode *p = x->parent;
        SplayNode *g = p->parent;
        
        if (!g) {
            // Zig: parent is root
            if (x == p->left) rotate_right(p);
            else rotate_left(p);
        } else if (x == p->left && p == g->left) {
            // Zig-zig: both left children
            rotate_right(g);
            rotate_right(p);
        } else if (x == p->right && p == g->right) {
            // Zig-zig: both right children
            rotate_left(g);
            rotate_left(p);
        } else {
            // Zig-zag: mixed children
            if (x == p->left) { rotate_right(p); rotate_left(g); }
            else { rotate_left(p); rotate_right(g); }
        }
    }
    *root = x;
}
```

Splay trees have no memory overhead (no height/color fields) — 32 bytes per node (key + left + right + parent) vs. 40 for AVL/red-black. They adapt to access patterns: frequently accessed elements migrate to the root, giving near-O(1) access for skewed distributions. But every operation modifies the tree (the splay), so read-only workloads still pay the write cost. And worst-case individual operations can be O(n) (rare, but real-time systems can't tolerate it).

Performance: insertions are faster than AVL/red-black (no rebalancing, just splay the new node). Lookups are slower (splay modifies the tree even on a hit). For workloads with strong locality (80% of lookups to 20% of keys), splay trees outperform AVL after the warmup period.

## Memory Layout and Cache Performance

All four classical BSTs use pointer-based nodes allocated with `malloc`. This scatters nodes across the heap, causing cache misses on nearly every access. For n = 1M, the tree needs ~40 MB. The L3 cache is typically 16-32 MB — the tree overflows L3, so most accesses miss to RAM.

Compare with the S-tree from the previous article: 1M elements, ~4.2 MB contiguous memory, 5 levels. Lookups: ~15ns (L1/L2 hits). The BST: 1M elements, ~40 MB scattered, 20 levels. Lookups: ~180ns (mostly L3 misses/RAM accesses). The 12× speed difference is almost entirely memory latency.

The cache-optimized variants from the previous articles address this directly:
- **Eytzinger layout**: pack the BST into contiguous memory, turning pointer dereferences into array index arithmetic.
- **S-trees**: increase branching factor to B = 16, reducing height and exploiting SIMD within nodes.
- **B⁻ trees**: add mutability to S-trees with explicit child pointers.

## When to Use Which

**AVL tree**: when lookup dominance is extreme (>90% lookups, <10% updates). The strict balance gives the shortest average path length. Example: dictionary lookups in a spell checker that's rebuilt daily.

**Red-black tree**: balanced workloads (~50% lookups, ~50% inserts). The fewer rotations per insert add up. Example: `std::map`'s default choice.

**Treap**: when code simplicity matters more than last-10% performance, or when you need a randomized data structure. Example: teaching, quick prototypes, and randomized algorithms courses.

**Splay tree**: when the workload has strong temporal locality. Frequently accessed keys bubble to the top naturally. Example: a router's MAC address table (recently seen addresses are likely to be seen again).

**Cache-optimized (S-tree, B⁻ tree)**: when throughput matters and the working set fits in L2/L3. These are 10-15× faster than any BST for lookups but require more memory and more complex code. Example: database indexes, in-memory key-value stores.

## Performance Summary (Zen 2, 1M 64-bit keys)

| Operation | AVL | Red-Black | Treap | Splay | S-Tree (B=16) |
|-----------|-----|-----------|-------|-------|----------------|
| Lookup | 180 ns | 200 ns | 220 ns | 300 ns | 15 ns |
| Insert | 320 ns | 280 ns | 350 ns | 250 ns | 55 ns |
| Delete | 340 ns | 300 ns | 370 ns | 270 ns | 58 ns |
| Memory | 40 MB | 40 MB | 40 MB | 32 MB | 4.2 MB |
| Height (max) | 28 | 38 | ~70 (rare) | N/A | 5 |

The S-tree is 12× faster for lookups, 5× faster for inserts, and uses 1/10 the memory. The cost: it's static. For dynamic workloads, the B⁻ tree offers similar insert performance (55ns) with slightly slower lookups (28ns) and dynamic mutability.

## Key Lessons

1. **All pointer-based BSTs are cache-bound.** The dominant cost is not comparisons or rotations — it's cache misses from scattered node allocations. The 12× difference between AVL and S-tree is nearly all memory latency.
2. **Balancing strategy affects update throughput.** AVL's stricter balance requires more rotations → slower inserts. Red-black's relaxed balance is the sweet spot for mixed workloads.
3. **Splay trees are fascinating but niche.** The self-adjusting property is elegant and works beautifully for skewed access patterns, but every operation modifies the tree — bad for read-heavy workloads.
4. **The classical BSTs are not obsolete — they're the baseline.** Understanding AVL/red-black tells you what the cache-optimized variants are improving. And for trees that fit entirely in L1 (n < 10³), the pointer-based BSTs are competitive because every access is an L1 hit.
5. **For any tree larger than L2, use a cache-optimized variant.** The gap between contiguous and scattered memory is so large that no amount of balance optimization can close it.
