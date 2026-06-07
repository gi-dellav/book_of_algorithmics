# Hash Tables

Hash tables are the most used data structure in software — Python's `dict`, JavaScript's objects, Go's `map`, C++'s `std::unordered_map`, and every database join algorithm depend on them. A well-designed hash table answers lookups in ~10ns; a poorly designed one in ~100ns. The difference is entirely in the implementation decisions: open addressing vs. chaining, probing strategy, load factor, and SIMD acceleration.

## The Design Space

A hash table maps keys to values with O(1) average-case operations. The core design decisions:

1. **Collision resolution**: what happens when two keys hash to the same slot?
2. **Probing strategy**: in what order do we search for an empty slot?
3. **Load factor**: how full can the table get before we resize?
4. **Hash function**: how do we turn keys into indices?
5. **Memory layout**: how are keys and values arranged in memory?

Each decision affects lookup latency, insertion throughput, memory overhead, and deletion complexity.

## Open Addressing vs. Chaining

**Chaining**: each bucket is a linked list (or balanced tree). Collisions append to the list. Used by `std::unordered_map`.

```c
struct ChainNode {
    Key key;
    Value value;
    ChainNode *next;
};
ChainNode *table[capacity];  // Array of linked list heads
```

Lookup: hash → bucket index → walk the chain. Insertion: hash → bucket → prepend to chain. Deletion: walk chain, unlink node.

Advantages: simple, handles high load factors gracefully (chains just grow), deletion is easy. Disadvantages: pointer chasing (each node is a separate allocation → cache misses), memory overhead (one pointer per node + allocation metadata), and no SIMD acceleration (sequential chain traversal).

**Open addressing**: all keys are stored directly in the table array. Collisions probe for the next available slot. Used by Python `dict`, Go `map`, Rust `HashMap`, and `absl::flat_hash_map`.

```c
struct Slot {
    Key key;
    Value value;
    bool occupied;  // Or use a sentinel key value
};
Slot table[capacity];
```

Lookup: hash → index. If `table[index].key == key`: found. If `table[index].occupied == false`: not found. Otherwise, probe to the next slot.

Open addressing has better cache behavior (contiguous array, no pointer chasing) and is 2-3× faster than chaining for most workloads. The disadvantage: performance degrades at high load factors (clustering).

## Probing Strategies

**Linear probing**: `h(k, i) = (h(k) + i) % capacity`. The simplest: just scan forward. Advantage: excellent cache behavior (sequential access after the first probe). Disadvantage: **primary clustering** — runs of occupied slots grow and merge, increasing probe lengths.

**Quadratic probing**: `h(k, i) = (h(k) + c₁·i + c₂·i²) % capacity`. Reduces primary clustering but introduces **secondary clustering** (keys that hash to the same initial slot follow the same probe sequence).

**Double hashing**: `h(k, i) = (h₁(k) + i · h₂(k)) % capacity` where h₂(k) is never 0 and is coprime to the capacity. Eliminates clustering entirely. But: two hash computations per probe, and the probe sequence jumps around in memory — worse cache behavior than linear probing.

**Robin Hood hashing**: during insertion, if the current probe length exceeds the probe length of the key already in the slot, swap them. The "richer" key (shorter probe distance) stays; the "poorer" key continues probing. This reduces the variance of probe lengths — all lookups have approximately the same cost. Used by Rust's `HashMap`.

Robin Hood insertion:

```c
void robin_hood_insert(Slot *table, int cap, Key key, Value val) {
    uint64_t h = hash(key);
    int idx = h % cap;
    int dist = 0;  // Distance from ideal position
    
    while (true) {
        if (!table[idx].occupied) {
            table[idx].key = key;
            table[idx].value = val;
            table[idx].occupied = true;
            return;
        }
        
        int existing_dist = (idx - (hash(table[idx].key) % cap) + cap) % cap;
        if (dist > existing_dist) {
            // Swap: the new key is further from its ideal position
            swap(table[idx].key, key);
            swap(table[idx].value, val);
            dist = existing_dist;  // Continue with the displaced key
        }
        
        idx = (idx + 1) % cap;
        dist++;
    }
}
```

Robin Hood lookup: the search stops when we encounter a key with shorter probe distance than our current distance — if the key were in the table, it would have been inserted before that point.

Performance: Robin Hood hashing has ~10% slower insertions (swaps) but ~20% faster lookups (reduced variance → fewer max-probe-length cases) compared to linear probing. The reduced variance is valuable for real-time systems where worst-case latency matters.

## SIMD-Accelerated Probing (Swiss Tables)

**Swiss tables** (used by `absl::flat_hash_map` and Rust's `hashbrown`) accelerate open addressing with SIMD. The key insight: most probes find an empty slot or a match within the first few tries. SIMD can check 16 slots in parallel.

The table is divided into **groups** of 16 slots. A separate **metadata** array stores a 1-byte hash prefix (the top 7 bits of the hash) per slot. A group's metadata is 16 bytes — one SSE register.

```c
struct SwissTable {
    Key keys[capacity];
    Value values[capacity];
    uint8_t metadata[capacity];  // Top 7 bits of hash; 0x80 = empty, 0x7F = tombstone
};
```

Lookup:

```c
Value swiss_lookup(SwissTable *t, int cap, Key key) {
    uint64_t h = hash(key);
    uint8_t h1 = h >> 57;  // Top 7 bits as the "signature"
    int group_idx = (h % cap) / 16;
    
    while (true) {
        // Load the metadata for this group of 16
        __m128i meta = _mm_loadu_si128((__m128i*)&t->metadata[group_idx * 16]);
        
        // Create a vector where each byte = h1
        __m128i target = _mm_set1_epi8(h1);
        
        // Compare: which slots have a matching signature?
        __m128i match = _mm_cmpeq_epi8(meta, target);
        int mask = _mm_movemask_epi8(match);
        
        if (mask != 0) {
            // Possible matches: check each candidate
            while (mask) {
                int bit = __builtin_ctz(mask);
                int idx = group_idx * 16 + bit;
                if (t->keys[idx] == key)
                    return t->values[idx];  // Found
                mask &= mask - 1;  // Clear lowest set bit
            }
        }
        
        // Check for empty slots (metadata byte = 0x80)
        __m128i empty = _mm_set1_epi8(0x80);
        __m128i is_empty = _mm_cmpeq_epi8(meta, empty);
        int empty_mask = _mm_movemask_epi8(is_empty);
        
        if (empty_mask != 0) {
            return NOT_FOUND;  // Key is not in the table
        }
        
        // Probe to the next group
        group_idx = (group_idx + 1) % (cap / 16);
    }
}
```

The SIMD comparison checks 16 slots in ~5 cycles (load + broadcast + compare + movemask + branch). A linear probing loop checks 1 slot in ~4 cycles (load + compare + branch). SIMD is ~3× faster per slot, but the slots are "wider" (checking 16 at a time). For load factors up to 87.5%, the average probe count is ~2 for Swiss tables — so ~2 SIMD comparisons = ~10 cycles → ~5ns per lookup.

Insertion follows the same pattern: find a group with an empty slot (using the metadata comparison), then find the specific empty slot within that group, and write the key, value, and metadata.

Deletion sets the metadata byte to `0x7F` (tombstone) instead of `0x80` (empty). Tombstones are treated as empty for insertion but not for lookup (they don't stop the search). Too many tombstones degrade performance; the table is rehashed when the tombstone fraction exceeds a threshold.

## Load Factor and Resizing

The **load factor** α = n / capacity. As α → 1, probe lengths grow:

- Linear probing: expected probes for a successful lookup ≈ ½(1 + 1/(1-α)). At α = 0.5: ~1.5 probes. At α = 0.9: ~5.5 probes. At α = 0.99: ~50 probes.
- Robin Hood: variance is lower (max probe length ≈ O(log n) instead of O(n)) but the average is similar.
- Swiss tables: the expected probes for a group is also ½(1 + 1/(1-α)), but each "probe" checks 16 slots. At α = 0.875 (7/8): ~4.5 groups = ~72 slots checked. But the first group has ~87.5% chance of containing the key or an empty slot.

Most implementations resize at α ≈ 0.75–0.875. The resize allocates a new table (2× capacity), rehashes all keys, and deallocates the old table. This is an O(n) operation, but amortized over n insertions, it adds O(1) per insertion.

Growth strategies:
- **Power-of-two capacity**: `capacity = 2^k`. The hash modulo capacity is just `hash & (capacity - 1)` — a bitwise AND, not a division. This is significantly faster. But it means the hash function must have good entropy in the lower bits.
- **Prime capacity**: `capacity` is a prime. Hash modulo prime uses all bits of the hash. Slower modulo operation but better distribution for weak hash functions.

Modern implementations use power-of-two capacity with a strong hash function (SipHash, HighwayHash, or even `fibonacci_hash` mixing).

## Hash Functions

The hash function must be:
1. **Fast**: a lookup does at least one hash computation. If hashing is slower than memory access, it's the bottleneck.
2. **Uniform**: keys should be distributed uniformly across buckets regardless of their distribution.
3. **Deterministic**: same key → same hash.

For integer keys: **Fibonacci hashing** (Knuth's multiplicative method):

```c
uint64_t fibonacci_hash(uint64_t key) {
    // Multiply by 2^64 / φ ≈ 11400714819323198485
    return key * 11400714819323198485ull;
}
```

The multiplication is 3 cycles on Zen 2. The top bits are the best-mixed; use `hash >> (64 - log2(capacity))` as the bucket index. For power-of-two capacity, just use the low bits of the Fibonacci hash.

For string keys: the hash function must mix bytes across the whole string. **SipHash-1-3** is the standard for hash table implementations (used in Rust, Python, Go) because it's fast (2 cycles per byte for short strings) and cryptographically resistant to hash-flooding attacks.

## Performance Summary (Zen 2, 64-bit integer keys and values, 1M entries)

| Implementation | Lookup | Insert | Memory/Entry | Max Load Factor |
|---------------|--------|--------|--------------|-----------------|
| `std::unordered_map` (chaining) | 85 ns | 120 ns | ~56 B | 1.0 |
| Linear probing | 28 ns | 45 ns | ~16 B | 0.75 |
| Robin Hood | 22 ns | 50 ns | ~16 B | 0.875 |
| Swiss table (SSE, group=16) | 12 ns | 30 ns | ~18 B | 0.875 |
| Swiss table (AVX2, group=32) | 9 ns | 25 ns | ~18 B | 0.875 |

The Swiss table is ~7× faster than `std::unordered_map` for lookups and uses ~3× less memory. The SSE metadata comparison (16 slots/instruction) is the key enabler. AVX2 (32 slots/instruction) provides a further ~25% improvement.

## Deletion and Tombstones

Open addressing complicates deletion. If we simply mark a slot as "empty," later probes that passed through that slot will stop prematurely and fail to find their key. The fix: **tombstones** — mark deleted slots with a special value (e.g., metadata = 0x7F in Swiss tables) that continues probes but allows insertion.

Tombstones accumulate. After many deletions, the table is mostly tombstones → probes are long but the table is "empty." The fix: rehash when the tombstone count exceeds a threshold. Most implementations rehash during resize, or explicitly when `tombstones > capacity / 4`.

An alternative: **backward shift deletion**. When deleting a key, look forward for a key that can be moved back to fill the gap without breaking probe sequences. This keeps the table tombstone-free but is more complex to implement.

## Key Lessons

1. **Open addressing beats chaining on modern hardware.** The cache misses from pointer chasing dominate. Contiguous arrays are 2-3× faster.
2. **SIMD transforms hash table probing.** Checking 16 (or 32) slots per instruction changes the economics: higher load factors are viable, probe count matters less, and throughput is gated by SIMD width.
3. **The hash function must be fast and uniform.** Fibonacci hashing for integers (3 cycles) and SipHash for strings are the modern defaults. Weak hash functions cause clustering that no probing strategy can fix.
4. **Tombstones are a necessary evil.** Deletion without tombstones is possible (backward shift) but the complexity is rarely worth it. Rehash when tombstone count exceeds a threshold.
5. **Swiss tables are the current state of the art.** `absl::flat_hash_map` and Rust's `hashbrown` both use the SIMD metadata approach. For most workloads, they are within 2× of the theoretically optimal hash table.
