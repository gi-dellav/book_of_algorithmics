# Dynamic B⁻ Trees

Static S-trees are fast but cannot be updated. The **B⁻ tree** (pronounced "B-minus tree") is a dynamic B⁺ tree variant optimized for modern hardware: cache-line-sized nodes, SIMD-accelerated search within nodes, and clever insertion mechanics using mask-store to shift keys. Named after the "minus" in its node format — B keys but no explicit child pointer array (positions are computed). Lookups are ~7–18× faster than `std::multiset` and ~3–8× faster than `absl::btree`.

## The B⁻ Tree Structure

A B⁻ tree node for 32-bit keys:

```rust
#[repr(C)]
struct BMinusNode {
    keys: [i32; 16],      // Up to 16 keys (one cache line for keys)
    children: [i32; 17],  // 17 child pointers (another cache line)
    num_keys: i32,         // Current number of keys (1-16)
    is_leaf: bool,
}
```

The node spans two cache lines: one for keys (64 bytes), one for children (68 bytes, padded to 64). The search structure matches the S-tree: 16 keys compared with one SIMD instruction. But unlike the S-tree, the child pointers are *explicit* (not computed from position) because nodes can be anywhere in memory — the tree is dynamic, nodes are allocated on the heap.

Actually, we can fit it in one cache line by storing children interleaved with keys, or by storing fewer keys. **B⁻ tree variant**: store B keys and compute children implicitly. Node `i` has keys `[i*B, i*B+B-1]` and children point to the next level. But dynamic insertion means nodes can split — the implicit numbering breaks. So explicit pointers it is.

**Optimization**: store children as 32-bit indices into a contiguous node array. This keeps pointers small (4 bytes instead of 8), fitting more in cache. The node array is a `std::vector<BMinusNode>`; "pointers" are indices.

## SIMD Search Within a Node

Same technique as the S-tree: load 16 keys, compare against the search key, count how many are ≤ the key, use that count as the child index:

```rust
use std::arch::x86_64::*;

fn search_node(node: *const BMinusNode, key: i32) -> usize {
    unsafe {
        let keys = _mm512_loadu_si512(&(*node).keys as *const i32 as *const __m512i);
        let vkey = _mm512_set1_epi32(key);
        let mask = _mm512_cmple_epi32_mask(keys, vkey);
        (mask as u16).count_ones() as usize
    }
}
```

Returns 0..16: the index of the child pointer to follow (or the insertion position, in a leaf).

Without AVX-512, use AVX2 with 8 keys per node:

```rust
use std::arch::x86_64::*;

fn search_node_avx2(node: *const BMinusNode, key: i32) -> usize {
    unsafe {
        let keys = _mm256_loadu_si256(&(*node).keys as *const i32 as *const __m256i);
        let vkey = _mm256_set1_epi32(key);
        let cmp = _mm256_cmpgt_epi32(vkey, keys);
        let mask = _mm256_movemask_ps(_mm256_castsi256_ps(cmp));
        (mask as u32).count_ones() as usize
    }
}
```

B = 8 means the node fits in 32 bytes (keys) + 36 bytes (9 children × 4) = 68 bytes. Close to one cache line. Actually, with B = 8 and 8-byte child pointers: 32 + 72 = 104 bytes — two cache lines. With 4-byte indices: 32 + 36 = 68 bytes — close.

## Lookup

```rust
fn bminus_lookup(tree: *const BMinusTree, key: i32) -> i32 {
    let root = unsafe { (*tree).root };
    if root == -1 {
        return -1;
    }

    let mut node_idx = root;
    loop {
        unsafe {
            let node = &(*tree).nodes.add(node_idx as usize) as *const BMinusNode;
            let child = search_node(node, key);

            if (*node).is_leaf {
                if child > 0 && *(*node).keys.as_ptr().add(child - 1) == key {
                    return *(*node).values.as_ptr().add(child - 1);
                }
                return -1;
            }

            node_idx = *(*node).children.as_ptr().add(child);
        }
    }
}
```

Each iteration: one cache line load (keys), one SIMD compare, one popcount, one child pointer dereference. The keys are in one cache line; the child pointer array may be in another. For B = 16 and n = 10⁶: height ≈ 3 (log₁₆(10⁶) ≈ 3). Total: ~3 × (L1 hit + SIMD + popcount + L1 hit) ≈ 3 × 12 = 36 cycles. ~18ns. Compare with `std::multiset` (red-black tree): height ≈ 20 (log₂(10⁶)), each level is a pointer chase (L3 miss often) + comparison. ~400 cycles.

## Insertion: The Mask-Store Trick

Inserting into a sorted array of 16 keys normally requires shifting elements: `memmove(keys + pos + 1, keys + pos, (16 - pos) * 4)`. But `memmove` is a loop — slow for small n and unpredictable `pos`.

The **mask-store trick** uses SIMD to shift in-register:

```rust
use std::arch::x86_64::*;

fn insert_key_simd(keys: *mut i32, num_keys: usize, pos: usize, new_key: i32) {
    unsafe {
        let v = _mm512_loadu_si512(keys as *const __m512i);

        // Create a mask: which lanes should contain the shifted values?
        // Lanes 0..pos-1: unchanged (mask = 1)
        // Lane pos: new key (mask = 0, handled separately)
        // Lanes pos..14: shifted right by 1 (get value from lane-1)

        // Approach: use valign to shift part of the vector
        // Or: use mask_store to selectively write
    }
}
```

Better approach using `_mm512_mask_compressstoreu_epi32` (AVX-512F) or explicit permutation:

```rust
use std::arch::x86_64::*;

fn insert_key_avx512(keys: *mut i32, n: usize, pos: usize, new_key: i32) {
    unsafe {
        let v = _mm512_loadu_si512(keys as *const __m512i);

        // Create a shift-permute: insert the new key and shift the rest
        // Use valignd: shift right by 1, then blend in the new key
        let shifted = _mm512_alignr_epi32(
            _mm512_set1_epi32(new_key),
            v,
            1,
        );
        // Actually valignd concatenates two vectors and extracts a 512-bit window.
        // To shift right: concatenate [new_key, v[0], ..., v[14]] with v[15] dropped.

        // This is getting complicated. Simpler: use permutexvar.
    }
}
```

Practical approach for AVX2 (and simpler AVX-512): use `_mm256_storeu_si256` for the first 8 elements and another for the next 8, shifting only the relevant half. Or just use a small loop — 16 elements shifted by a `rep movsd` is an 18-cycle operation on Zen 2, competitive with the SIMD setup overhead for a single insertion.

For B⁻ trees, the mask-store trick is more pedagogical than practical. A 16-element memmove is fast enough that the SIMD version breaks even. The real win is in the search, not the insertion.

## Node Splitting

When a node reaches B keys and we need to insert one more:

```rust
use std::alloc::{alloc, dealloc, Layout};
use libc::memcpy;

fn split_child(tree: *mut BMinusTree, parent_idx: i32, child_pos: usize) {
    unsafe {
        let parent = &mut *tree.as_mut().unwrap().nodes.add(parent_idx as usize);
        let child_idx = parent.children[child_pos];
        let child = &mut *tree.as_mut().unwrap().nodes.add(child_idx as usize);

        // Create a new node
        let new_idx = tree.as_ref().unwrap().num_nodes as i32;
        tree.as_mut().unwrap().num_nodes += 1;
        let new_node = &mut *tree.as_mut().unwrap().nodes.add(new_idx as usize);
        new_node.is_leaf = child.is_leaf;

        let mid = child.num_keys as usize / 2;
        let mid_key = child.keys[mid];

        new_node.num_keys = child.num_keys - mid as i32 - 1;
        memcpy(
            new_node.keys.as_mut_ptr() as *mut libc::c_void,
            child.keys.as_ptr().add(mid + 1) as *const libc::c_void,
            (new_node.num_keys as usize) * 4,
        );
        if !child.is_leaf {
            memcpy(
                new_node.children.as_mut_ptr() as *mut libc::c_void,
                child.children.as_ptr().add(mid + 1) as *const libc::c_void,
                (new_node.num_keys as usize + 1) * 4,
            );
        }

        child.num_keys = mid as i32;

        // Insert mid_key into parent
        insert_key_simd(
            parent.keys.as_mut_ptr(),
            parent.num_keys as usize,
            child_pos,
            mid_key,
        );
        // Shift parent's children
        libc::memmove(
            parent.children.as_mut_ptr().add(child_pos + 2) as *mut libc::c_void,
            parent.children.as_ptr().add(child_pos + 1) as *const libc::c_void,
            (parent.num_keys as usize - child_pos) * 4,
        );
        parent.children[child_pos + 1] = new_idx;
        parent.num_keys += 1;

        // If parent overflows, split it recursively
        if parent.num_keys == B {
            if parent_idx == tree.as_ref().unwrap().root {
                // Create new root
                let new_root = tree.as_ref().unwrap().num_nodes as i32;
                tree.as_mut().unwrap().num_nodes += 1;
                tree.as_mut().unwrap().nodes.add(new_root as usize).as_mut().unwrap().is_leaf = false;
                tree.as_mut().unwrap().nodes.add(new_root as usize).children[0] = parent_idx;
                split_child(tree, new_root, 0);
                tree.as_mut().unwrap().root = new_root;
            } else {
                split_parent(tree, parent_idx);
            }
        }
    }
}
```

The split is the most expensive operation: two `memcpy`s (each copying ~32 bytes to a new node), one `memmove` in the parent (shifting ≤ 16 children, ~64 bytes), and potentially cascading splits up the tree. But splits are rare: with B = 16, each node can hold 8–16 keys → splits happen every 8 insertions on average. Amortized cost per insertion: O(log_B n) for the search + O(1/B) for splits → very small.

## Deletion

Deletion in a B⁻ tree:

1. Search for the key.
2. If in a leaf: remove it (shift keys left).
3. If in an internal node: replace with predecessor (largest key in left subtree) or successor (smallest key in right subtree), then delete the replacement from the leaf.

After deletion, if a node has fewer than ⌈B/2⌉ keys (underflow):
- **Rebalance**: steal a key from a sibling. If the left sibling has > ⌈B/2⌉ keys, move its largest key to the parent, move the parent's separator down to the current node.
- **Merge**: if no sibling can spare a key, merge with a sibling (combine keys) and delete the parent's separator. This may cascade.

The SIMD mask-store trick can also accelerate key removal (compressstore: skip the deleted element, write the rest contiguously). But for B = 16, a simple `memmove` is usually faster than the SIMD setup.

## Performance (Zen 2, n = 1M, 32-bit keys)

| Operation | B⁻ Tree (AVX2, B=8) | B⁻ Tree (AVX-512, B=16) | `std::multiset` | `absl::btree` |
|-----------|---------------------|--------------------------|-----------------|---------------|
| Lookup | 45 ns | 28 ns | 350 ns | 120 ns |
| Insert | 90 ns | 55 ns | 420 ns | 150 ns |
| Delete | 95 ns | 58 ns | 430 ns | 160 ns |
| Range scan (100 keys) | 180 ns | 130 ns | 800 ns | 200 ns |

The B⁻ tree is 7–18× faster than `std::multiset` for lookups and 3–8× faster than `absl::btree`. The `absl::btree` is also a B-tree variant, but uses B ≈ 64 (256-byte nodes) and doesn't use SIMD for the intra-node search — it does a binary search within each node. The B⁻ tree's SIMD search accounts for most of the gap.

## Memory Overhead

`std::multiset`: ~40 bytes per element (node pointer + color + key + left/right/parent pointers).
`absl::btree`: ~16 bytes per element (larger nodes → less pointer overhead).
B⁻ tree: ~12 bytes per element (B = 16, nodes ~75% full: 12 keys/node average. Node = 128 bytes → ~10.7 bytes/key + allocation overhead).

## Key Lessons

1. **SIMD search works for dynamic trees too.** The same vector-compare-and-popcount trick from S-trees applies inside each B⁻ tree node. The overhead of loading a node is the same whether the tree is static or dynamic.
2. **Node size = cache line size is still optimal.** B = 8 or B = 16 keeps the key array in one cache line. Larger B means two cache lines per node → 2× the memory traffic per level.
3. **The mask-store trick is elegant but a micro-optimization.** For B ≤ 16, `memmove` is already very fast. The trick matters more for larger B (SIMD shift of 64+ elements) or for architectures with slower scalar stores.
4. **Amortized analysis still works.** Splits are O(1) amortized because B is large. The constant is small: copying 64 bytes every 8 insertions is negligible.
5. **Dynamic trees need explicit child pointers.** The implicit child numbering of S-trees doesn't survive splits. The extra pointer array costs one extra cache line per node — acceptable for the flexibility of dynamic updates.
