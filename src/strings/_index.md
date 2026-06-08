# Strings Are Not Just Characters

String algorithms suffer from a curious blind spot in algorithm textbooks: they're treated as abstract sequence problems, analyzed by counting character comparisons. On real hardware, string operations are bounded by two things: branch mispredictions (from character-by-character loops) and memory bandwidth (from scanning long strings). SIMD changes the game entirely — comparing 32 characters in one instruction.

## The String Matching Landscape

Given a pattern P of length m and a text T of length n (n ≫ m), find all occurrences of P in T. The classic algorithms (Knuth-Morris-Pratt, Boyer-Moore) achieve O(n) time by skipping characters that can't match. But on modern hardware, a naive SIMD-accelerated scan often beats them for m < 64 — it has no branches, no skip table lookups, and runs at memory bandwidth.

For longer patterns, we enter the territory of suffix arrays, FM-indexes, and compressed full-text indexes — data structures that preprocess the text for O(m) or O(log n) queries. These are the backbone of bioinformatics (aligning DNA reads to a reference genome) and full-text search engines.

## What This Chapter Covers

1. **Exact Pattern Matching** — Naive → SIMD (SSE4.2 `PCMPESTRI`, AVX2 masked compare) → KMP/Boyer-Moore → Horspool. When each wins, measured.
2. **Multi-Pattern Matching** — Aho-Corasick automaton, SIMD multi-pattern with `VPERM` shuffles, the Rabin-Karp rolling hash as a filter.
3. **Approximate String Matching** — Edit distance (Levenshtein) with SIMD, bit-parallel algorithms (Myers' bit-vector algorithm), k-mer matching for bioinformatics.
4. **Suffix Arrays and LCP** — Construction (SA-IS, prefix doubling), LCP with Kasai's algorithm, and why you should just use `libdivsufsort`.
5. **FM-Index and Burrows-Wheeler** — The BWT transform, backward search, wavelet trees for rank queries. The data structure behind Bowtie, BWA, and most DNA aligners.

## Recommended Reading Order

Start with **Exact Pattern Matching** — it establishes the SIMD comparison primitives used everywhere else. Then **Multi-Pattern Matching** for the Aho-Corasick/SIMD hybrid. Approximate matching and suffix arrays can be read independently.

Cross-reference Chapter 10 (SIMD) for the intrinsics, Chapter 6 (Arithmetic) for the rolling hash math, and Chapter 3 (Pipelining) for why branchless string loops matter.
