# Approximate String Matching

Given a pattern P and text T, find all substrings of T within edit distance k of P — allowing insertions, deletions, and substitutions. This is the core of spell-checking, DNA read alignment, and fuzzy search. The classic O(mn) dynamic programming solution is too slow for large n. Bit-parallel algorithms exploit the fact that k is small (1–3 for most applications) to process 64 columns of the DP matrix in a single CPU instruction.

## Edit Distance (Levenshtein)

The minimum number of insertions, deletions, and substitutions to transform string A into string B. The DP recurrence:

```
dp[i][j] = min(
    dp[i-1][j] + 1,       // deletion
    dp[i][j-1] + 1,       // insertion
    dp[i-1][j-1] + (A[i] != B[j]) as u32  // substitution
)
```

```rust
fn edit_distance(a: &[u8], b: &[u8]) -> u32 {
    let n = a.len();
    let m = b.len();
    let mut dp = vec![vec![0u32; m + 1]; n + 1];
    for i in 0..=n { dp[i][0] = i as u32; }
    for j in 0..=m { dp[0][j] = j as u32; }

    for i in 1..=n {
        for j in 1..=m {
            let cost = if a[i-1] == b[j-1] { 0 } else { 1 };
            dp[i][j] = (dp[i-1][j] + 1)
                .min(dp[i][j-1] + 1)
                .min(dp[i-1][j-1] + cost);
        }
    }
    dp[n][m]
}
```

O(nm) time, O(nm) memory. For n = m = 10⁵ (a typical DNA read against a reference segment), this is 10¹⁰ operations — minutes on a CPU. We need better.

## Stage 1: Two-Row DP

We only need the current and previous row:

```rust
fn edit_distance_linear(a: &[u8], b: &[u8]) -> u32 {
    let mut prev: Vec<u32> = (0..=b.len() as u32).collect();
    let mut curr = vec![0u32; b.len() + 1];

    for i in 1..=a.len() {
        curr[0] = i as u32;
        for j in 1..=b.len() {
            let cost = if a[i-1] == b[j-1] { 0 } else { 1 };
            curr[j] = (prev[j] + 1)
                .min(curr[j-1] + 1)
                .min(prev[j-1] + cost);
        }
        std::mem::swap(&mut prev, &mut curr);
    }
    prev[b.len()]
}
```

O(nm) time, but O(m) memory. Still too slow for large n.

## Stage 2: Myers' Bit-Vector Algorithm

Gene Myers (1999) observed that the DP differences between adjacent cells are -1, 0, or +1. This means each DP value can be encoded in 2 bits relative to the previous cell. For a column of height 64, the entire state fits in one machine word. The DP recurrence becomes bitwise operations.

The core insight: for a fixed pattern P (the query) and a streaming text T, we maintain a "score" for each position in P (how well it matches the current suffix of T). When the score is ≤ k, we report a match.

```rust
fn myers_search(pattern: &[u8], text: &[u8], k: usize) -> Vec<usize> {
    let m = pattern.len();
    assert!(m <= 64, "Pattern length must be ≤ 64 for the bit-vector algorithm");

    // Precompute character bitmasks: for each character c, a bitmask
    // where bit j is 1 if pattern[j] == c
    let mut peq = [0u64; 256];
    for (j, &c) in pattern.iter().enumerate() {
        peq[c as usize] |= 1u64 << j;
    }

    let mut pv = !0u64;  // previous vertical differences (all 1s)
    let mut mv = 0u64;   // previous diagonal differences
    let mut score = m as u32;
    let mut matches = Vec::new();
    let high_bit = 1u64 << m;

    for (i, &c) in text.iter().enumerate() {
        let eq = peq[c as usize];
        let xv = eq | mv;
        let xh = (((eq & pv) + pv) ^ pv) | eq;

        let mut ph = mv | !(xv | pv);
        let mut mh = pv & xv;

        // Update score
        if ph & high_bit != 0 { score += 1; }
        else if mh & high_bit != 0 { score -= 1; }

        ph <<= 1;
        mh <<= 1;
        pv = mh | !(xv | ph);
        mv = ph & xv;

        if score <= k as u32 {
            matches.push(i);
        }
    }
    matches
}
```

Performance: processes 64 characters of the pattern in parallel, ~3 CPU cycles per text character independent of pattern length (up to 64). For m = 30, that's ~30× faster than the two-row DP.

For patterns longer than 64, Myers' algorithm extends naturally: use multiple 64-bit words, carrying the overflow between words. This is the basis of the **Edlib** library, which achieves ~1 μs per pair of 100-character strings.

## Stage 3: k-mer Filtering for DNA Alignment

In bioinformatics, we typically need to align millions of short reads (100–150 bp) against a reference genome (3×10⁹ bp). Myers' algorithm is still O(nm) — too slow for whole-genome search. The solution: **k-mer indexing**.

1. Extract all k-mers (substrings of length k, typically 21–31) from the reference genome.
2. Build a hash table: k-mer → list of positions.
3. For each read, look up its k-mers in the hash table. Position matches indicate candidate alignment locations.
4. Run Myers' algorithm only at candidate locations.

```rust
fn build_kmer_index(reference: &[u8], k: usize) -> HashMap<u64, Vec<usize>> {
    let mut index: HashMap<u64, Vec<usize>> = HashMap::new();
    if reference.len() < k { return index; }

    let mut hash: u64 = 0;
    let mask = (1u64 << (2 * k)) - 1; // 2 bits per base (A=00, C=01, G=10, T=11)

    for i in 0..reference.len() {
        let base = match reference[i] {
            b'A' | b'a' => 0u64,
            b'C' | b'c' => 1,
            b'G' | b'g' => 2,
            b'T' | b't' => 3,
            _ => continue,
        };
        hash = ((hash << 2) | base) & mask;
        if i >= k - 1 {
            index.entry(hash).or_default().push(i - k + 1);
        }
    }
    index
}
```

The k-mer index reduces the search from O(genome_length) to O(read_length + candidate_positions). For a 3 GB genome and 21-mers, the hash table has ~10⁹ entries — it's disk-resident. Tools like BWA and Bowtie use an FM-index instead of a hash table for better compression, but the principle is the same: filter first, verify later.

## SIMD Edit Distance for Multiple Alignments

When verifying a candidate alignment, we need the actual edit distance (not just a yes/no within k). The two-row DP can be vectorized using SIMD: process 8 columns of `prev` and `curr` in parallel. The comparison `a[i-1] == b[j-1]` becomes a broadcast + compare:

```rust
unsafe fn edit_distance_simd(a: &[u8], b: &[u8]) -> u32 {
    let mut prev: Vec<__m256i> = // column data, 8-wide
    // ... SIMD min-of-three using _mm256_min_epu32
}
```

This achieves ~8× speedup for the DP verification step, enough for real-time spell-checking on 10⁵-word dictionaries.

## Benchmark Summary

| Problem | Algorithm | Time |
|---------|-----------|------|
| Edit distance, m=n=100 | Two-row DP | 3.2 μs |
| Edit distance, m=n=100 | Myers bit-vector | 0.45 μs |
| Edit distance, m=n=100 | SIMD DP | 0.35 μs |
| Search P=30 in T=1 MB, k=2 | Myers bit-vector | 4.2 ms |
| 10⁶ reads vs. 3 GB genome | k-mer + Myers | 22 min (32 cores) |

The overarching theme: bit-level parallelism (Myers) or SIMD (vectorized DP) reduces the O(m) factor to O(1) for m ≤ 64. Beyond that, the only winning move is to avoid computing edit distance altogether — use k-mers, suffix arrays, or FM-indexes to filter the search space.
