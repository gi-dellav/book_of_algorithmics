# Cache Eviction Policies

When the cache is full and a new block must be brought in, which block gets evicted? The choice of eviction policy affects the miss rate, sometimes dramatically. Real CPUs use approximations to LRU; this article covers the theory and practice.

## Common Policies

### FIFO (First-In, First-Out)

Evict the block that was brought in earliest. Simple to implement (a queue). Doesn't distinguish between frequently-used and rarely-used blocks. A block that's accessed every iteration may be evicted just because it was loaded first.

### LRU (Least Recently Used)

Evict the block that hasn't been accessed for the longest time. Intuitively: the past predicts the future. A block that was accessed recently is likely to be accessed again soon (temporal locality).

LRU is the gold standard in practice. Hardware caches approximate it with pseudo-LRU (binary tree of MRU bits, or a clock algorithm) because true LRU requires tracking access order for every cache line — expensive in hardware.

**LRU implementation**: Doubly-linked list + hash table. Every access moves the block to the front. Eviction removes from the back. This is O(1) per access but requires two pointers per block — fine for software, costly for hardware.

### LFU (Least Frequently Used)

Evict the block with the lowest access count. Good for workloads where popularity is stable (some blocks are "hot," others "cold"). Bad when a block was hot historically but won't be accessed again — it takes a long time for its frequency to decay.

### LIFO (Last-In, First-Out)

Evict the most recently loaded block. Terrible for most workloads (sequential scans evict the data they just loaded). Exists only as a strawman.

### MRU (Most Recently Used)

Evict the most recently accessed block. Surprisingly good for some sequential workloads: after reading a block, you may never need it again (streaming data). LRU keeps streaming data and evicts useful data; MRU evicts the streaming data immediately.

### RR (Random Replacement)

Pick a random block to evict. Simple to implement in hardware. Approximately as good as LRU for large caches with high associativity, but worse for small caches.

### Bélády's OPT (Optimal)

Evict the block that will be accessed furthest in the future. Requires knowing the entire future access sequence — impossible in practice, but provides a theoretical lower bound on miss rate. OPT is used to evaluate how good other policies are.

## The Sleator-Tarjan Theorem (1985)

**Theorem**: LRU never uses more than 2× the number of cache misses of OPT (for caches with at least twice the capacity of the OPT cache).

More precisely: LRU with cache size k has at most as many misses as OPT with cache size k/2 (for the same access sequence, assuming both start empty).

This is a "competitive analysis" result. It means LRU is within a constant factor of optimal, which is remarkable given that OPT knows the future and LRU doesn't. The practical implication: you can reason about OPT's behavior (which is easier to analyze theoretically) and know that LRU will be at most 2× worse (if you double the cache size) or within the same order.

## Real CPU Cache Policies

Modern CPUs use pseudo-LRU approximations:

- **Tree-PLRU**: A binary tree of bits, one per set. Each access updates the bits on the path to the accessed line. Eviction follows the opposite direction of the bits. Approximates LRU with much less hardware.
- **Quad-Age LRU** (Intel): For each cache line, 2 age bits. On access, age is reset to 0. On eviction, the oldest (age = 3) is chosen. Periodically, all ages are incremented.
- **RRIP** (Re-Reference Interval Prediction, Intel since Ivy Bridge): Each line has a "re-reference prediction value." Lines loaded with a predicted-near-immediate re-reference get low RRPV; others get high RRPV. Eviction picks high RRPV. Better than LRU for mixed-access workloads.

## Impact on Algorithms

The eviction policy affects which algorithmic patterns work well:

- **LRU-friendly**: Repeated scans of working sets that fit in cache. After the first scan, all data is in cache and subsequent scans hit.
- **LRU-hostile**: Scanning a large array once, then never again. The scan fills the cache with data that won't be reused, evicting data that would have been useful. MRU would handle this better.
- **Random-friendly**: Algorithms that touch a working set much larger than cache but with uniform random access. LRU thrashes (always evicts the wrong thing). Random replacement might do better, but avoiding random access entirely is the correct solution.
- **Optimal-friendly**: Algorithms with well-defined phases. The programmer knows the access pattern; the cache doesn't. Explicit prefetching and cache-bypass instructions (non-temporal stores) let the programmer override the policy.

## The Takeaway

You can't control the CPU's eviction policy, but you can design algorithms that are robust to it:
- Favor sequential access over random (works well with LRU and hardware prefetching).
- Keep working sets cache-sized.
- Use non-temporal stores for write-only data that won't be reused soon.
- Use software prefetching to hint the cache about upcoming accesses.
