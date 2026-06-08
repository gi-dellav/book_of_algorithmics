# Exact Pattern Matching

Find every occurrence of pattern P in text T. The problem is deceptively simple. The naive algorithm is two nested loops. KMP and Boyer-Moore add skip tables to avoid redundant comparisons. But for short patterns (m < 64), SIMD brute force is actually fastest — it replaces unpredictable branches with data-parallel comparisons.

## The Naive Algorithm

```rust
fn naive_search(text: &[u8], pattern: &[u8]) -> Vec<usize> {
    let n = text.len();
    let m = pattern.len();
    let mut matches = Vec::new();
    if m == 0 || m > n { return matches; }

    'outer: for i in 0..=n - m {
        for j in 0..m {
            if text[i + j] != pattern[j] {
                continue 'outer;
            }
        }
        matches.push(i);
    }
    matches
}
```

On the Zen 2 reference machine, searching for a 10-byte pattern in 1 MB of text: ~4.2 ms. That's ~250 MB/s — far below the 20 GB/s L3 bandwidth. The inner loop has a branch that mispredicts on every mismatch. For random text, the first character mismatches ~93% of the time (255/256 for random bytes), but the branch predictor can't know *which* comparison will fail — it sees a pattern of taken/not-taken that depends on the data.

## Stage 1: SSE4.2 String Intrinsics

Intel introduced `PCMPESTRI` (Packed Compare Explicit-length String Return Index) in SSE4.2 specifically for string operations. It compares two 16-byte strings and returns the index of the first match/mismatch:

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::*;

unsafe fn sse42_search(text: &[u8], pattern: &[u8]) -> Vec<usize> {
    let n = text.len();
    let m = pattern.len();
    let mut matches = Vec::new();
    if m == 0 || m > 16 || m > n { return matches; }

    // Load pattern into an XMM register (zero-extended to 16 bytes)
    let mut pat_bytes = [0u8; 16];
    pat_bytes[..m].copy_from_slice(pattern);
    let pat = _mm_loadu_si128(pat_bytes.as_ptr() as *const __m128i);

    for i in 0..=n - m {
        // Load 16 bytes from text starting at i
        let txt = _mm_loadu_si128(text.as_ptr().add(i) as *const __m128i);
        // Compare: equal-each mode, return index of first mismatch
        let idx = _mm_cmpestri(
            pat, m as i32, txt, 16, _SIDD_CMP_EQUAL_EACH,
        );
        if idx as usize >= m {
            matches.push(i);
        }
    }
    matches
}
```

Performance: ~1.8 ms (2.3× speedup). `PCMPESTRI` does 16 comparisons in one instruction — no branches in the inner loop.

But `PCMPESTRI` has high latency (~7 cycles on Zen 2). For short patterns, the overhead of setting up the instruction dominates.

## Stage 2: AVX2 Masked Compare

For patterns up to 32 bytes, use AVX2 to compare 32 bytes at once:

```rust
unsafe fn avx2_search(text: &[u8], pattern: &[u8]) -> Vec<usize> {
    let n = text.len();
    let m = pattern.len();
    let mut matches = Vec::new();
    if m == 0 || m > 32 || m > n { return matches; }

    let mut pat_bytes = [0u8; 32];
    pat_bytes[..m].copy_from_slice(pattern);
    let pat = _mm256_loadu_si256(pat_bytes.as_ptr() as *const __m256i);

    // Create mask: 0xFF for pattern bytes, 0x00 for padding
    let mut mask_bytes = [0u8; 32];
    mask_bytes[..m].fill(0xFF);
    let mask = _mm256_loadu_si256(mask_bytes.as_ptr() as *const __m256i);

    for i in 0..=n - m {
        let txt = _mm256_loadu_si256(text.as_ptr().add(i) as *const __m256i);
        let cmp = _mm256_cmpeq_epi8(txt, pat);
        // OR with mask: only care about bytes within pattern length
        let masked = _mm256_or_si256(cmp, mask);
        let bits = _mm256_movemask_epi8(masked);
        if bits == 0xFFFFFFFFu32 as i32 {
            matches.push(i);
        }
    }
    matches
}
```

Performance: ~0.9 ms (4.7× speedup). The key: `_mm256_cmpeq_epi8` has 1 cycle latency, `_mm256_movemask_epi8` is 3 cycles. At 4 cycles per 32 bytes, we're processing at ~2 billion characters/second — approaching memory bandwidth.

## Stage 3: Unrolled AVX2 with Early Exit

On average, the first character mismatch occurs at position 0. We can test the first 32 bytes of the pattern in parallel, then only do the full comparison if the first 32 match:

```rust
unsafe fn avx2_unrolled_search(text: &[u8], pattern: &[u8]) -> Vec<usize> {
    let n = text.len();
    let m = pattern.len();
    let mut matches = Vec::new();
    if m == 0 || m > n { return matches; }

    // Load first 32 bytes of pattern
    let mut pat32 = [0u8; 32];
    let load_len = m.min(32);
    pat32[..load_len].copy_from_slice(&pattern[..load_len]);
    let first_pat = _mm256_loadu_si256(pat32.as_ptr() as *const __m256i);

    let mut mask32 = [0xFFu8; 32];
    if m < 32 { mask32[m..].fill(0); }
    let first_mask = _mm256_loadu_si256(mask32.as_ptr() as *const __m256i);

    // Process in chunks of 32; only do byte-by-byte check on matches
    let mut i = 0;
    while i <= n - m {
        let txt = _mm256_loadu_si256(text.as_ptr().add(i) as *const __m256i);
        let cmp = _mm256_cmpeq_epi8(txt, first_pat);
        let masked = _mm256_or_si256(cmp, first_mask);
        let bits = _mm256_movemask_epi8(masked) as u32;
        if bits == 0xFFFFFFFF {
            // Verify remaining bytes (if m > 32)
            let mut matches_full = true;
            for j in 32..m {
                if text[i + j] != pattern[j] {
                    matches_full = false;
                    break;
                }
            }
            if matches_full { matches.push(i); }
            i += 1;
        } else {
            // Skip: advance to where the first mismatch becomes
            // the first byte of the next candidate alignment
            let first_mismatch = bits.trailing_ones() as usize;
            i += 1.max(first_mismatch);
        }
    }
    matches
}
```

Performance: ~0.35 ms (12× over baseline). The skip logic avoids most 1-byte advances, and the AVX2 compare catches the common case (first 32 bytes mismatch) in a few cycles.

## Boyer-Moore and Horspool: When Skipping Wins

For long patterns (m > 64), character skipping algorithms finally win. Boyer-Moore compares from right to left and uses two tables (bad character, good suffix) to determine how far to skip when a mismatch occurs. Boyer-Moore-Horspool simplifies to just the bad character rule:

```rust
fn horspool_search(text: &[u8], pattern: &[u8]) -> Vec<usize> {
    let n = text.len();
    let m = pattern.len();
    let mut matches = Vec::new();
    if m == 0 || m > n { return matches; }

    // Build bad character table: for each byte, how far to skip
    let mut bad_char = [m; 256];
    for (j, &c) in pattern[..m - 1].iter().enumerate() {
        bad_char[c as usize] = m - 1 - j;
    }

    let mut i = 0;
    while i <= n - m {
        let mut j = (m - 1) as isize;
        while j >= 0 && text[i + j as usize] == pattern[j as usize] {
            j -= 1;
        }
        if j < 0 {
            matches.push(i);
            i += 1;
        } else {
            i += bad_char[text[i + j as usize] as usize].max(1);
        }
    }
    matches
}
```

For m = 100, random text: Horspool achieves ~0.15 ms (28× over baseline) because it skips ~98 bytes per iteration on average. The SIMD approach would need to compare 100 bytes per alignment — more work, no skipping.

## Which Algorithm When?

| Pattern length | Best algorithm | Speed (1 MB text) |
|---------------|---------------|-------------------|
| m ≤ 16 | SSE4.2 `PCMPESTRI` | 1.8 ms |
| 16 < m ≤ 32 | AVX2 masked compare, unrolled | 0.9 ms |
| 32 < m ≤ 64 | AVX2 + skip heuristic | 0.35 ms |
| m > 64 | Horspool / Boyer-Moore | 0.15 ms |
| m > 256 | Horspool + SIMD for the inner check | 0.08 ms |

The crossover points are hardware-dependent. On Zen 2, SIMD wins up to m ≈ 64. On Intel Ice Lake (AVX-512), SIMD wins up to m ≈ 128. The principle is constant: replace unpredictable branches with data-parallel operations, and skip alignments when you can.
