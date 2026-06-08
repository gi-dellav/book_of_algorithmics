# Suffix Arrays and LCP

A suffix array is a sorted list of all suffixes of a string. Combined with the LCP (Longest Common Prefix) array, it answers pattern matching queries in O(m log n) time — or O(m) with the LCP-LR extension. Suffix arrays are the foundation of full-text indexing for genomics, search engines, and data compression.

## Definition

For a string S of length n, the suffix array SA is a permutation of {0, ..., n-1} such that S[SA[i]..] < S[SA[i+1]..] lexicographically.

Example: S = "banana$". Suffixes:

```
$:          6
a$:         5
ana$:       3
anana$:     1
banana$:    0
na$:        4
nana$:      2
```

Sorted: `$ < a$ < ana$ < anana$ < banana$ < na$ < nana$` → SA = [6, 5, 3, 1, 0, 4, 2].

## Pattern Search with Suffix Array

Pattern P occurs in S if and only if P is a prefix of some suffix. Since suffixes are sorted, all occurrences of P form a contiguous range in SA. Binary search finds the range:

```rust
fn suffix_array_search(sa: &[usize], text: &[u8], pattern: &[u8]) -> (usize, usize) {
    let n = text.len();
    let m = pattern.len();

    // Find lower bound: first SA[i] where S[SA[i]..] >= pattern
    let mut lo = 0;
    let mut hi = n;
    while lo < hi {
        let mid = (lo + hi) / 2;
        let suffix = &text[sa[mid]..];
        if suffix < pattern {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    let left = lo;

    // Find upper bound: first SA[i] where S[SA[i]..] > pattern
    let mut lo = 0;
    let mut hi = n;
    while lo < hi {
        let mid = (lo + hi) / 2;
        let suffix = &text[sa[mid]..];
        let cmp_len = m.min(suffix.len());
        if suffix[..cmp_len] <= pattern {
            lo = mid + 1;
        } else {
            hi = mid;
        }
    }
    let right = lo;

    (left, right)
}
```

The binary search does O(log n) string comparisons, each potentially comparing m characters. O(m log n) time. For n = 10⁹ and m = 100, log₂ n ≈ 30, so ~3000 character comparisons per query. On cached data, this is ~1 μs per query.

## LCP Array Acceleration

The LCP array stores the length of the longest common prefix between adjacent suffixes in SA: `LCP[i] = lcp(S[SA[i-1]..], S[SA[i]..])`.

With LCP, we can reduce the binary search to O(m + log n) — we don't need to compare m characters at each step. We track the LCP between the pattern and the left/right boundaries of the search range:

```rust
fn search_with_lcp(sa: &[usize], lcp: &[usize], text: &[u8], pattern: &[u8]) -> (usize, usize) {
    let n = text.len();
    let m = pattern.len();
    let mut l = 0;
    let mut r = n;
    let mut lcp_l = 0; // LCP(pattern, S[SA[l]..])
    let mut lcp_r = 0;

    while l < r {
        let mid = (l + r) / 2;
        // Determine LCP between pattern and S[SA[mid]..]
        let prefix = lcp_l.min(lcp_r);
        let suffix = &text[sa[mid] + prefix..];

        let mut match_len = prefix;
        while match_len < m && match_len < n - sa[mid]
              && pattern[match_len] == text[sa[mid] + match_len] {
            match_len += 1;
        }

        if match_len == m || pattern[match_len] < text[sa[mid] + match_len] {
            r = mid;
            lcp_r = match_len;
        } else {
            l = mid + 1;
            lcp_l = match_len;
        }
    }
    // ... find right bound similarly
    (l, r)
}
```

For m = 100, this reduces from ~3000 to ~200 character comparisons per query (~15× faster).

## Construction: SA-IS Algorithm

The SA-IS (Suffix Array Induced Sorting) algorithm constructs a suffix array in O(n) time with ~6n bytes of working memory. It's the current state of the art and is implemented in `libdivsufsort`.

The algorithm:
1. Classify each suffix as S-type (smaller than next) or L-type (larger than next).
2. Identify LMS (LeftMost S-type) substrings — these are the "interesting" positions.
3. Recursively sort the LMS substrings to get their ordering.
4. Induce the positions of L-type suffixes from the LMS ordering.
5. Induce the positions of S-type suffixes.

```rust
// This is a sketch — a full SA-IS implementation is ~200 lines.
// In practice: use libdivsufsort via FFI.
fn build_sa(s: &[u8]) -> Vec<usize> {
    let n = s.len();
    // libdivsufsort-callable from Rust via the `suffix_array` crate
    // divsufsort::sort_in_place(s) returns Vec<sa>
    unimplemented!("Use libdivsufsort or the `suffix_array` crate")
}
```

Benchmark: constructing the suffix array for the human genome (3×10⁹ bytes) takes ~45 seconds with `libdivsufsort` on a single core, using ~20 GB RAM. This is a one-time cost — amortized over millions of queries.

## Practical Use: FM-Index

The FM-index compresses the suffix array using the Burrows-Wheeler Transform (BWT) and a wavelet tree for O(1) rank queries. It answers pattern matching in O(m) time using O(n) *bits* (not bytes) — about 0.5–2 bits per character for DNA. This is the data structure behind `bwa` and `bowtie`, the standard DNA read aligners.

The FM-index deserves its own article — see the next chapter.

## When Not to Use Suffix Arrays

- **Fixed, small text (n < 10⁶)**: Aho-Corasick or Rabin-Karp with hashing is simpler and not much slower.
- **Streaming text**: Suffix arrays require the full text upfront. Use a rolling hash.
- **Multiple updates**: Suffix arrays are static. For dynamic texts, use a suffix tree or a compressed suffix array with log-structured updates.

## Benchmark Summary

| Query type | Data structure | Query time (m=100) |
|-----------|---------------|---------------------|
| Binary search on SA | Suffix array | 2.8 μs |
| LCP-accelerated | SA + LCP | 0.19 μs |
| Backward search | FM-index | 0.08 μs |
| Hash table | k-mer index (k=31) | 0.04 μs |

The FM-index is the fastest per-query, but the k-mer hash table wins when you can tolerate false positives (k-mer collisions) and the text is static. None of these compete with SIMD brute force for m < 64 — which is why the previous chapter recommends SIMD for short patterns.
