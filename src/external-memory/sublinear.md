# Sublinear Algorithms

Sublinear algorithms process data without looking at all of it. For massive datasets where even a linear scan is too expensive, sublinear algorithms — sampling, sketching, and streaming — provide approximate answers with provable guarantees. This article surveys the key techniques.

## When Linear Is Too Slow

If you have 10⁹ records and need to answer a query in 1 ms, you can't scan 10⁹ elements (at 4 bytes/element, that's 4 GB — even at 100 GB/s bandwidth, that's 40 ms). You need an algorithm that runs in time sublinear in N — typically O(log N), O(1), or O(1/ε²).

These algorithms are "sketching" or "streaming" algorithms: they process data in one pass, maintain a small summary (the "sketch"), and answer queries from the sketch.

## Reservoir Sampling

Pick k random elements from a stream of unknown length, where each element has equal probability k/N of being selected.

```rust
let mut reservoir = vec![0; k];
// Fill initial reservoir
for i in 0..k {
    reservoir[i] = stream[i];
}

// Replace with decreasing probability
for i in k..N {
    let j = (unsafe { libc::rand() } as usize) % (i + 1);
    if j < k {
        reservoir[j] = stream[i];
    }
}
```

At step i, each element so far has probability k/i of being in the reservoir. After processing all N elements, each element has probability k/N. Proof by induction.

Reservoir sampling uses O(k) memory and O(N) time (but only one pass). Used when you need a random sample from a dataset too large to shuffle or index.

## Count-Min Sketch

Approximate the frequency of each element in a stream. Uses O((1/ε) × log(1/δ)) space where ε is the error bound and δ is the failure probability.

```
Count-Min Sketch:
  2D array count[depth][width]
  d hash functions h_1, ..., h_depth
  
  Update(item, delta):
    for i = 1 to depth:
      count[i][h_i(item) % width] += delta
  
  Query(item):
    return min(count[i][h_i(item) % width]) for i = 1..depth
```

The estimate is always ≥ true frequency (one-sided error). With probability 1−δ, the error is at most ε × sum of all frequencies. For width = ⌈e/ε⌉ and depth = ⌈ln(1/δ)⌉.

Count-Min Sketch uses kilobytes to estimate frequencies of millions of distinct items. Used in network monitoring (heavy hitters), database query planning, and NLP (word frequency estimation).

## HyperLogLog

Estimate the number of distinct elements in a stream (cardinality). Uses ~1.5 KB of memory, independent of the actual cardinality. Typical error: 2%.

**Idea**: Hash each element. Count the maximum number of leading zeros in the hashed values. If the maximum is k, then roughly 2^k distinct elements were seen (because the probability of observing k leading zeros is 1/2^k).

**HyperLogLog** (Flajolet et al., 2007) refines this with:
- Multiple registers (2^precision) for better accuracy.
- Harmonic mean instead of geometric mean (reduces outlier influence).
- Bias correction for small and large cardinalities.

```rust
// Simplified HyperLogLog
struct HyperLogLog {
    registers: [u32; 65536],  // precision = 16 (typical)
}

impl HyperLogLog {
    fn add(&mut self, hash: u64) {
        let idx = (hash & 0xFFFF) as usize;  // First 16 bits: register index
        let zeros = (hash | 0xFFFF).leading_zeros() + 1;  // Count leading zeros
        self.registers[idx] = self.registers[idx].max(zeros);
    }

    fn estimate(&self) -> f64 {
        let mut sum = 0.0;
        for i in 0..65536 {
            sum += 1.0 / ((1u64 << self.registers[i]) as f64);
        }
        let raw = 0.7213 * 65536.0 * 65536.0 / sum;  // Alpha_16 * m^2 / sum
        // Bias correction for small/large ranges...
        raw
    }
}
```

Used in Redis (PFADD/PFCOUNT), Spark, BigQuery, and every analytics system that needs to count distinct users, page views, or events.

## Bloom Filters (Preview)

A Bloom filter answers: "Is this item in the set?" with possible false positives (says yes when the answer is no) but no false negatives. Covered in detail in `data-structures/filters.md`.

- k hash functions, m-bit array.
- `add(x)`: set bits h_1(x), ..., h_k(x) to 1.
- `query(x)`: return true if all bits h_1(x), ..., h_k(x) are 1.

False positive rate: (1 − e^(−kn/m))^k. Optimal k = (m/n) × ln(2). For 1% false positive rate, use ~9.6 bits per element.

Memory: 1 GB can store a Bloom filter for ~800 million elements with 1% false positive rate — vs. 8 GB for a raw set of 64-bit integers.

## The Streaming Model

Streaming algorithms process data as a sequence, with severely limited memory (typically O(log^k N) or O(1/ε²)). They cannot store the entire input. They answer queries about the data after a single pass (or occasionally a few passes).

Problems solvable in the streaming model:
- **Frequency moments**: Count-Min Sketch (F_1 — frequencies), Count Sketch (F_2 — second moment).
- **Distinct elements**: HyperLogLog (F_0).
- **Heavy hitters**: Items with frequency > φN. Space: O(1/φ).
- **Order statistics**: Approximate median, quantiles. Space: O(1/ε).
- **Graph problems**: Sparsification, spectral approximation, triangle counting.

Problems NOT solvable in the streaming model with sublinear space:
- Exact set membership (Bloom filters have false positives).
- Shortest path in a graph.
- Clustering with quality guarantees.

## Practical Guidance

1. **Sublinear algorithms give approximate answers.** The error bounds are probabilistic. For most analytics and monitoring, 2% error is acceptable. For billing and financial reporting, it's not.

2. **The constants matter.** Count-Min Sketch with ε=1% and δ=0.01% needs ~2,700 counters. That's ~11 KB — trivial. But for ε=0.01%, it needs ~270,000 counters (~1 MB). The space grows as 1/ε, so precision is expensive.

3. **Use library implementations.** HyperLogLog and Count-Min Sketch have subtle implementation details (hash function quality, bias corrections, merge semantics). Use Redis, Apache DataSketches, or Akka's implementations.

4. **Combine with sampling.** If you need to store a sample for further analysis, use reservoir sampling to capture a uniform sample, then analyze that sample offline.
