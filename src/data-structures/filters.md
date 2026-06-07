# Bloom Filters

A Bloom filter is a probabilistic set: it tells you either "definitely not in the set" or "possibly in the set." False negatives are impossible; false positives have a configurable probability. The payoff: a Bloom filter uses ~1 byte per element regardless of element size. For a cache of 100 million URLs, that's 100 MB instead of 10+ GB. Bloom filters are the inverse of a cache — caches store a subset of data precisely; Bloom filters represent a superset approximately.

## How It Works

A Bloom filter is a bit array of m bits and k independent hash functions. To insert an element: compute k hash values, set those k bits to 1. To query: compute the same k hash values. If all k bits are 1 → "possibly present." If any bit is 0 → "definitely absent."

```c
struct BloomFilter {
    uint64_t *bits;
    int m;  // Number of bits
    int k;  // Number of hash functions
};

void bloom_add(BloomFilter *bf, uint64_t key) {
    uint64_t h = key;
    for (int i = 0; i < bf->k; i++) {
        h = murmur64(h);  // Re-hash to get independent hash values
        int idx = h % bf->m;
        bf->bits[idx >> 6] |= 1ULL << (idx & 63);
    }
}

bool bloom_query(BloomFilter *bf, uint64_t key) {
    uint64_t h = key;
    for (int i = 0; i < bf->k; i++) {
        h = murmur64(h);
        int idx = h % bf->m;
        if (!((bf->bits[idx >> 6] >> (idx & 63)) & 1))
            return false;  // Definitely not present
    }
    return true;  // Possibly present
}
```

## False Positive Rate

After inserting n elements into a Bloom filter with m bits and k hash functions, the probability that a specific bit is still 0 is:

p₀ = (1 - 1/m)^(k·n) ≈ e^(-k·n/m)

The false positive probability (all k bits happen to be 1 for an uninserted element):

ε = (1 - p₀)^k ≈ (1 - e^(-k·n/m))^k

For fixed m and n, the optimal k minimizes ε:

k_opt = (m/n) · ln(2) ≈ 0.693 · m/n

At this optimum: ε ≈ (0.6185)^(m/n). To achieve ε = 1%: m/n ≈ 9.6 bits per element, k ≈ 6.6 → k = 7. To achieve ε = 0.1%: m/n ≈ 14.4 bits, k ≈ 10.

Memory for 100M elements at ε = 1%: 100M × 9.6 bits = 120 MB. At ε = 0.1%: 180 MB. Compare with 100M × 8 bytes (URL pointers) = 800 MB minimum for a hash set.

## Double Hashing

Computing k independent hash functions is expensive. **Double hashing** generates all k indices from two base hashes:

```
h₁ = hash1(key)
h₂ = hash2(key)
idx_i = (h₁ + i · h₂) mod m   for i = 0, 1, ..., k-1
```

This is provably as good as k independent hash functions for Bloom filters (Kirsch & Mitzenmacher, 2006). Performance: 2 hash computations instead of k. For k = 7: ~3.5× faster.

```c
void bloom_add_double_hash(BloomFilter *bf, uint64_t key) {
    uint64_t h1 = hash1(key);
    uint64_t h2 = hash2(key) | 1;  // Ensure h2 is never 0 (so indices cycle through all m)
    for (int i = 0; i < bf->k; i++) {
        int idx = (h1 + i * h2) % bf->m;
        bf->bits[idx >> 6] |= 1ULL << (idx & 63);
    }
}
```

## Blocked Bloom Filters (Cache-Efficient)

A standard Bloom filter scatters k bits randomly across m bits. Each query causes k random memory accesses — cache misses if the filter is larger than L3. A **blocked Bloom filter** divides the filter into blocks of one cache line (512 bits = 64 bytes) and ensures all k bits for a single element fall within the *same* block.

```c
struct BlockedBloom {
    uint64_t blocks[MAX_BLOCKS][8];  // Each block: 8 × 64 bits = 512 bits = 1 cache line
    int num_blocks;
    int k;  // Number of bits per element (must fit within 512)
};

void blocked_bloom_add(BlockedBloom *bf, uint64_t key) {
    uint64_t h = hash(key);
    int block_idx = h % bf->num_blocks;  // Which block
    uint64_t block_hash = hash2(key);
    
    for (int i = 0; i < bf->k; i++) {
        int bit = (block_hash + i * PRIME) % 512;
        bf->blocks[block_idx][bit >> 6] |= 1ULL << (bit & 63);
    }
}
```

Each query touches exactly one cache line. For filters larger than L3, the blocked version is ~3× faster. The tradeoff: restricted to k bits within 512 bits → maximum k ≈ 30 (for ε well below 0.01%). For typical k = 7, this is no constraint.

## Cuckoo Filters

**Cuckoo filters** (Fan et al., 2014) improve on Bloom filters in two ways: they support deletion (Bloom filters can't — removing a bit might remove another element's bit), and they have better space efficiency for low false positive rates.

A cuckoo filter stores **fingerprints** (short hashes) of elements in a hash table with cuckoo hashing: each element has two candidate buckets, h₁(key) and h₂(key) = h₁(key) ⊕ hash(fingerprint). To insert: try both buckets; if both are full, evict a random existing element and re-insert it in its alternate bucket. This may cascade, but the expected number of displacements is O(1) for load factors below 50%.

```c
struct CuckooFilter {
    uint8_t *buckets;        // Each bucket holds 4 fingerprints
    int bucket_size;         // 4 entries per bucket
    int num_buckets;
    int fingerprint_bits;    // 8-16 bits per fingerprint
};

void cuckoo_insert(CuckooFilter *cf, uint64_t key) {
    uint8_t fp = fingerprint(key, cf->fingerprint_bits);
    uint32_t h1 = hash(key) % cf->num_buckets;
    uint32_t h2 = h1 ^ hash_fp(fp);
    
    // Try to insert in bucket h1 or h2
    if (bucket_insert(cf, h1, fp)) return;
    if (bucket_insert(cf, h2, fp)) return;
    
    // Both full: evict and re-insert
    uint32_t bucket = (rand() & 1) ? h2 : h1;
    for (int i = 0; i < MAX_DISPLACEMENTS; i++) {
        int slot = rand() % cf->bucket_size;
        uint8_t evicted = cf->buckets[bucket * cf->bucket_size + slot];
        cf->buckets[bucket * cf->bucket_size + slot] = fp;
        fp = evicted;
        bucket = bucket ^ hash_fp(fp);  // Alternate bucket for evicted fingerprint
        
        if (bucket_insert(cf, bucket, fp)) return;
    }
    // Failed: resize and retry
}
```

A cuckoo filter with 12-bit fingerprints and 4 entries per bucket achieves ε = 0.0003 (0.03%) at 8.5 bits per element — better than a Bloom filter's 14.4 bits for the same ε. And it supports deletion: just remove the fingerprint.

The downside: insertions can fail (unlikely but possible). In practice, resizing the filter on failure is acceptable for most use cases. Lookup is O(1) with exactly 2 bucket accesses — faster than a Bloom filter's k random accesses.

## Applications

**LevelDB/RocksDB**: Bloom filters at each SSTable level to avoid disk seeks for non-existent keys. Without a Bloom filter, every `Get(key)` reads from every level. With one, 99% of negative lookups are filtered at memory speed (~50ns) instead of disk speed (~10ms).

**Cassandra**: Bloom filters on SSTables to avoid I/O on key lookups. Configurable false positive rate per column family.

**Chrome**: Bloom filter of malicious URLs. The full list is gigabytes; the Bloom filter is megabytes and ships with the browser. A positive check triggers a network lookup to the full list.

**Networking**: IP address filtering, packet routing. A Bloom filter in an FPGA or switch can filter packets at line rate (100+ Gbps) because it's just k bit checks per packet.

## Performance Summary (Zen 2, 100M elements, ε = 1%)

| Filter | Add | Query | Memory | Deletion? |
|--------|-----|-------|--------|-----------|
| Bloom (7 hashes) | 45 ns | 40 ns | 120 MB | No |
| Bloom (double hash) | 18 ns | 15 ns | 120 MB | No |
| Blocked Bloom | 12 ns | 10 ns | 120 MB | No |
| Cuckoo (12-bit fp) | 35 ns | 14 ns | 96 MB | Yes |
| Cuckoo (8-bit fp) | 30 ns | 12 ns | 80 MB | Yes |

## Key Lessons

1. **Bloom filters trade certainty for space.** At 9.6 bits per element, you get 1% false positive rate. No hash set can match this — the minimum for a 64-bit hash set is 64 bits per element.
2. **Double hashing eliminates the k-hash bottleneck.** Two hashes generate k indices through simple arithmetic. Provably as good as independent hashes for Bloom filters.
3. **Blocked Bloom filters are cache-friendly.** Constraining k bits to one cache line ensures one memory access per query. Critical for filters that don't fit in L3.
4. **Cuckoo filters add deletion and better space at low ε.** For ε < 0.1%, cuckoo filters use fewer bits per element and support deletion. The tradeoff: insertion can fail (rarely).
5. **Bloom filters are an inverse cache.** A cache stores a precise subset; a Bloom filter stores an approximate superset. Both use hashing, both have limited capacity, both improve the performance of a larger, slower backing store.
