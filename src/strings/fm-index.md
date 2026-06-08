# FM-Index and Burrows-Wheeler

The FM-index (Ferragina-Manzini, 2000) is a compressed full-text index that answers pattern matching in O(m) time while using less space than the original text. It's the most important data structure in bioinformatics — every major DNA read aligner (BWA, Bowtie, SOAP2) uses it. This article explains how it works and why it's fast.

## Burrows-Wheeler Transform

The BWT of a string S is the last column of the sorted list of all cyclic rotations of S. Convention: append a sentinel `$` (smaller than all characters) to S, then sort the rotations:

S = "banana$". Rotations sorted lexicographically:

```
$banana
a$banan
ana$ban
anana$b
banana$
na$bana
nana$ba
```

BWT = last column = "annb$aa".

Key property: the BWT is **reversible**. Given only the BWT and the position of `$`, you can reconstruct the original string. And it tends to group identical characters together — "annb$aa" has runs of 'a' and 'n' — making it highly compressible with run-length encoding.

## Backward Search

The FM-index searches for a pattern by iterating from right to left, maintaining the range of SA indices where the current suffix matches. This is called **backward search**:

```rust
struct FMIndex {
    bwt: Vec<u8>,              // Burrows-Wheeler transform
    c_table: [usize; 256],     // C[c] = number of characters < c in S
    occ: Box<dyn RankQuery>,   // occ[c][i] = number of occurrences of c in BWT[0..i]
}

impl FMIndex {
    fn backward_search(&self, pattern: &[u8]) -> (usize, usize) {
        let mut lo = 0usize;
        let mut hi = self.bwt.len();

        for &c in pattern.iter().rev() {
            let c = c as usize;
            lo = self.c_table[c] + self.occ.rank(c, lo);
            hi = self.c_table[c] + self.occ.rank(c, hi);
            if lo >= hi { break; } // pattern not found
        }
        (lo, hi)
    }
}
```

Example: search for "ana" in "banana$":

1. Start: lo=0, hi=7 (all suffixes).
2. c = 'a': C['a'] = 1 (only '$' before 'a'). rank('a', 0) = 0, rank('a', 7) = 3. lo=1, hi=4.
3. c = 'n': C['n'] = 4. rank('n', 1) = 0, rank('n', 4) = 2. lo=4, hi=6.
4. c = 'a': C['a'] = 1. rank('a', 4) = 2, rank('a', 6) = 3. lo=3, hi=5.

Result: SA[3..5] = {3, 1} — suffixes "ana$" and "anana$", both starting with "ana".

## Rank Queries with Wavelet Trees

The `rank(c, i)` operation — "how many times does character c appear in BWT[0..i)?" — must be fast. The standard solution is a **wavelet tree**:

A wavelet tree is a binary tree over the alphabet. Each node stores a bitmap indicating which characters go to the left child (0) and which to the right (1). To compute rank(c, i), traverse from root to leaf, using the bitmaps to refine the position:

```rust
struct WaveletTree {
    bitmap: Vec<u64>,     // packed bits: 0 = left child, 1 = right
    // For each node: popcount prefix sums for O(1) rank on bitmaps
    left_child: Option<Box<WaveletTree>>,
    right_child: Option<Box<WaveletTree>>,
    lo_char: u8,
    hi_char: u8,
}

impl WaveletTree {
    fn rank(&self, c: u8, i: usize) -> usize {
        if self.lo_char == self.hi_char { return i; }

        let mid = (self.lo_char + self.hi_char) / 2;
        if c <= mid {
            // Character is in left child. How many characters ≤ mid
            // are in BWT[0..i)? That's i - rank(1, i) on the bitmap.
            let new_i = i - self.bitmap_rank1(i);
            self.left_child.as_ref().map_or(0, |lc| lc.rank(c, new_i))
        } else {
            let new_i = self.bitmap_rank1(i);
            self.right_child.as_ref().map_or(0, |rc| rc.rank(c, new_i))
        }
    }

    fn bitmap_rank1(&self, i: usize) -> usize {
        let word = i / 64;
        let bit = i % 64;
        let mask = (1u64 << bit) - 1;
        (self.prefix_sum[word] + (self.bitmap[word] & mask).count_ones()) as usize
    }
}
```

The bitmap rank uses `popcnt` (3 cycles) on the partial word, plus a precomputed prefix sum. Total: ~5 cycles per rank query per level. With alphabet size 256 (8 levels per query) and pattern length m, backward search takes ~40m cycles — independent of text size.

## Compression

The FM-index compresses the BWT using run-length encoding + entropy coding. For DNA (4-character alphabet), the BWT typically compresses to 0.5–2 bits per base. The wavelet tree's bitmaps add ~0.5 bits per base. Total: ~1–2.5 bits per base, or 300–750 MB for the human genome (3×10⁹ bases). The original text is 750 MB (2 bits per base, packed). So the FM-index is actually *smaller* than the raw text for DNA.

## Practical FM-Index from Rust

```rust
use suffix_array::SuffixArray;

struct SimpleFM {
    bwt: Vec<u8>,
    c_table: [usize; 256],
    occ_sparse: Vec<[u32; 256]>, // rank sampled every 128 positions
    sample_sa: Vec<usize>,        // SA sampled every 32 positions
}

impl SimpleFM {
    fn build(text: &[u8]) -> Self {
        let n = text.len();
        let sa = SuffixArray::new(text);

        // Compute BWT
        let bwt: Vec<u8> = (0..n).map(|i| {
            if sa[i] == 0 { text[n - 1] } else { text[sa[i] - 1] }
        }).collect();

        // C table
        let mut c_table = [0usize; 256];
        for &c in text.iter() { c_table[c as usize + 1] += 1; }
        for c in 1..256 { c_table[c] += c_table[c - 1]; }

        // Sparse rank samples (every 128 positions)
        let sample_step = 128;
        let occ_sparse: Vec<[u32; 256]> = (0..=(n / sample_step))
            .map(|i| {
                let pos = (i * sample_step).min(n);
                let mut counts = [0u32; 256];
                for &c in &bwt[..pos] { counts[c as usize] += 1; }
                counts
            })
            .collect();

        // SA samples (every 32 positions)
        let sa_step = 32;
        let sample_sa: Vec<usize> = (0..n)
            .step_by(sa_step)
            .map(|i| sa[i])
            .collect();

        SimpleFM { bwt, c_table, occ_sparse, sample_sa }
    }

    fn rank(&self, c: u8, i: usize) -> usize {
        let sample = i / 128;
        let base = self.occ_sparse[sample][c as usize] as usize;
        let start = sample * 128;
        base + self.bwt[start..i].iter().filter(|&&x| x == c).count()
    }
}
```

For production use, the rank sampling step size should be tuned: smaller steps = faster queries but more memory. A step of 128 gives ~2% memory overhead with <0.1 μs query time on cached data.

## Benchmark Summary (pattern search in 3 GB human genome)

| Method | Memory | Query time (m=100) |
|--------|--------|---------------------|
| Raw suffix array | 12 GB | 0.19 μs |
| FM-index (sparse rank) | 800 MB | 0.08 μs |
| FM-index (wavelet tree) | 400 MB | 0.12 μs |
| k-mer hash table (k=31) | 8 GB | 0.04 μs |

The FM-index with wavelet tree is the sweet spot: it's smaller than the raw genome (compressed) and answers queries at CPU speed. For most bioinformatics applications, this is the only data structure you need.
