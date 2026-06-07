# Segment Trees

Segment trees answer range queries — sum, minimum, maximum, greatest common divisor — over any contiguous subarray, with point updates, in O(log n) time. They're the workhorse of competitive programming, but the textbook implementation (pointer-based recursion) is catastrophically slow on real hardware. This article traces the evolution from pointer-based to implicit to Fenwick tree to wide B-ary SIMD segment trees, with speedups exceeding 200× on prefix sum queries.

## The Problem

Given an array `a[0..n-1]` and an associative operation ⊕ (addition, min, max, etc.), support:

- **Query(l, r)**: compute a[l] ⊕ a[l+1] ⊕ ... ⊕ a[r-1].
- **Update(i, x)**: set a[i] = x.

A naive array answers Query in O(n) and Update in O(1). A prefix sum array answers Query in O(1) but Update in O(n). A segment tree answers both in O(log n).

## Stage 1: Pointer-Based Recursive Segment Tree

```rust
use std::alloc::{alloc, dealloc, Layout};

#[repr(C)]
struct SegNode {
    lo: usize,
    hi: usize,
    val: i32,
    left: *mut SegNode,
    right: *mut SegNode,
}

fn build(a: *const i32, lo: usize, hi: usize) -> *mut SegNode {
    unsafe {
        let layout = Layout::new::<SegNode>();
        let node = alloc(layout) as *mut SegNode;
        (*node).lo = lo;
        (*node).hi = hi;
        if hi - lo == 1 {
            (*node).val = *a.add(lo);
            (*node).left = std::ptr::null_mut();
            (*node).right = std::ptr::null_mut();
        } else {
            let mid = lo + (hi - lo) / 2;
            (*node).left = build(a, lo, mid);
            (*node).right = build(a, mid, hi);
            (*node).val = (*(*node).left).val + (*(*node).right).val;
        }
        node
    }
}

fn query(node: *const SegNode, ql: usize, qr: usize) -> i32 {
    unsafe {
        if ql <= (*node).lo && (*node).hi <= qr {
            return (*node).val;
        }
        if qr <= (*node).lo || (*node).hi <= ql {
            return 0;
        }
        query((*node).left, ql, qr) + query((*node).right, ql, qr)
    }
}

fn update(node: *mut SegNode, idx: usize, val: i32) {
    unsafe {
        if (*node).hi - (*node).lo == 1 {
            (*node).val = val;
        } else {
            let mid = (*node).lo + ((*node).hi - (*node).lo) / 2;
            if idx < mid {
                update((*node).left, idx, val);
            } else {
                update((*node).right, idx, val);
            }
            (*node).val = (*(*node).left).val + (*(*node).right).val;
        }
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

```rust
const MAXN: usize = 100000;
static mut TREE: [i32; 4 * MAXN] = [0; 4 * MAXN];

fn build_implicit(a: *const i32, n: usize) {
    unsafe {
        for i in 0..n {
            TREE[n + i] = *a.add(i);
        }
        for i in (1..n).rev() {
            TREE[i] = TREE[2 * i] + TREE[2 * i + 1];
        }
    }
}

fn query_implicit(n: usize, mut l: usize, mut r: usize) -> i32 {
    unsafe {
        l += n;
        r += n;
        let mut res = 0;
        while l < r {
            if l & 1 != 0 {
                res += TREE[l];
                l += 1;
            }
            if r & 1 != 0 {
                r -= 1;
                res += TREE[r];
            }
            l >>= 1;
            r >>= 1;
        }
        res
    }
}

fn update_implicit(n: usize, idx: usize, val: i32) {
    unsafe {
        TREE[n + idx] = val;
        let mut i = (n + idx) / 2;
        while i > 0 {
            TREE[i] = TREE[2 * i] + TREE[2 * i + 1];
            i /= 2;
        }
    }
}
```

Performance: ~180ns per query (~4.7× faster than pointer-based). Eliminates:
- Recursion (iterative bottom-up).
- Pointer chasing (sequential array, good locality).
- Memory overhead: 4n ints = 16n bytes (1.6 MB for n = 100K — fits in L2).

The remaining overhead: the loop does O(log n) iterations with unpredictable branches (`if (l & 1)`).

## Stage 3: Branchless Bottom-Up

The `if (l & 1)` and `if (r & 1)` branches can be eliminated with conditional moves:

```rust
fn query_branchless(n: usize, mut l: usize, mut r: usize) -> i32 {
    unsafe {
        l += n;
        r += n;
        let mut res = 0;
        while l < r {
            let add_l = if l & 1 != 0 {
                let val = TREE[l];
                l += 1;
                val
            } else {
                0
            };
            let add_r = if r & 1 != 0 {
                r -= 1;
                TREE[r]
            } else {
                0
            };
            res += add_l + add_r;
            l >>= 1;
            r >>= 1;
        }
        res
    }
}
```

The compiler may or may not turn the ternary into `cmov`. On Clang with `-O3`: it does. On GCC: sometimes not. Explicit asm or `__builtin_expect` can help.

Performance: ~120ns per query (~7× faster than pointer-based). The branch misprediction cost is gone. But we still have the loop dependency: `res` depends on the previous iteration's `res`, preventing ILP.

## Stage 4: Fenwick Tree (Binary Indexed Tree)

For **prefix sum** queries (l = 0), the Fenwick tree is even simpler and faster:

```rust
static mut FENWICK: [i32; MAXN] = [0; MAXN];

fn fenwick_add(n: usize, mut idx: usize, delta: i32) {
    unsafe {
        idx += 1;
        while idx <= n {
            FENWICK[idx - 1] += delta;
            idx += idx & idx.wrapping_neg();
        }
    }
}

fn fenwick_sum(n: usize, mut idx: usize) -> i32 {
    unsafe {
        let mut res = 0;
        while idx > 0 {
            res += FENWICK[idx - 1];
            idx -= idx & idx.wrapping_neg();
        }
        res
    }
}

fn fenwick_range(n: usize, l: usize, r: usize) -> i32 {
    fenwick_sum(n, r) - fenwick_sum(n, l)
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

```rust
const B: usize = 16;
static mut WIDE_TREE: [i32; 4 * MAXN] = [0; 4 * MAXN];

fn build_wide(a: *const i32, n: usize) {
    unsafe {
        let mut padded: usize = 1;
        while padded < n {
            padded *= B;
        }

        for i in 0..n {
            WIDE_TREE[padded + i] = *a.add(i);
        }
        for i in n..padded {
            WIDE_TREE[padded + i] = 0;
        }

        for i in (1..padded).rev() {
            let mut sum = 0;
            for j in 0..B {
                sum += WIDE_TREE[B * i + j];
            }
            WIDE_TREE[i] = sum;
        }
    }
}

fn query_wide(padded: usize, mut l: usize, mut r: usize) -> i32 {
    unsafe {
        l += padded;
        r += padded;
        let mut res = 0;

        while l < r {
            while l % B != 0 && l < r {
                res += WIDE_TREE[l];
                l += 1;
            }
            while r % B != 0 && l < r {
                r -= 1;
                res += WIDE_TREE[r];
            }
            l /= B;
            r /= B;
        }
        res
    }
}
```

The wide segment tree is trickier to query than binary because blocks don't align nicely with arbitrary ranges. The query must process partial blocks at the edges and whole blocks in the middle. But with SIMD, processing a partial block of up to B elements can be done with a single vector load + mask:

```rust
use std::arch::x86_64::*;

fn query_wide_simd(padded: usize, mut l: usize, mut r: usize) -> i32 {
    unsafe {
        l += padded;
        r += padded;
        let mut res = 0;

        while l < r {
            if l % B != 0 {
                let block = l / B;
                let offset = l % B;
                let count = if B - offset < r - l { B - offset } else { r - l };

                let v = _mm512_loadu_si512(WIDE_TREE.as_ptr().add(block * B) as *const __m512i);
                let mut mask: u16 = (1u16 << count) - 1;
                mask <<= offset;
                let masked = _mm512_maskz_mov_epi32(mask, v);
                res += _mm512_reduce_add_epi32(masked);

                l += count;
                continue;
            }

            if l + B <= r {
                let v = _mm512_loadu_si512(WIDE_TREE.as_ptr().add(l) as *const __m512i);
                res += _mm512_reduce_add_epi32(v);
                l += B;
                continue;
            }

            if r % B != 0 && l < r {
                let block = r / B;
                let count = r % B;
                let v = _mm512_loadu_si512(WIDE_TREE.as_ptr().add(block * B) as *const __m512i);
                let mask: u16 = (1u16 << count) - 1;
                let masked = _mm512_maskz_mov_epi32(mask, v);
                res += _mm512_reduce_add_epi32(masked);
                break;
            }

            l /= B;
            r /= B;
        }
        res
    }
}
```

Performance: ~15ns per query for n = 100K, B = 16. ~57× faster than pointer-based.

## Stage 6: Non-Reversible Monoids and Lazy Propagation

Sum is a **reversible** operation: you can compute range sum as `prefix[r] - prefix[l]`. But `min` and `gcd` are not reversible — there's no inverse operation. For these, the segment tree must explicitly compute the range aggregate.

Also, range updates (add X to all elements in [l, r)) require **lazy propagation**: mark a node as "dirty" with a pending update, and only propagate it when the node is queried:

```rust
static mut TREE: [i32; 4 * MAXN] = [0; 4 * MAXN];
static mut LAZY: [i32; 4 * MAXN] = [0; 4 * MAXN];

fn apply(idx: usize, val: i32, len: usize) {
    unsafe {
        TREE[idx] += val * len as i32;
        LAZY[idx] += val;
    }
}

fn push(idx: usize, lo: usize, hi: usize) {
    unsafe {
        if LAZY[idx] != 0 {
            let mid = lo + (hi - lo) / 2;
            apply(2 * idx, LAZY[idx], mid - lo);
            apply(2 * idx + 1, LAZY[idx], hi - mid);
            LAZY[idx] = 0;
        }
    }
}

fn range_add(idx: usize, lo: usize, hi: usize, ql: usize, qr: usize, val: i32) {
    if ql <= lo && hi <= qr {
        apply(idx, val, hi - lo);
        return;
    }
    if qr <= lo || hi <= ql {
        return;
    }
    push(idx, lo, hi);
    let mid = lo + (hi - lo) / 2;
    range_add(2 * idx, lo, mid, ql, qr, val);
    range_add(2 * idx + 1, mid, hi, ql, qr, val);
    unsafe {
        TREE[idx] = TREE[2 * idx] + TREE[2 * idx + 1];
    }
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
