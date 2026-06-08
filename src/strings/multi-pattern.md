# Multi-Pattern Matching

Given a set of k patterns, find all occurrences of any pattern in the text. The Aho-Corasick automaton solves this in O(n + total matches) time, independent of k. It's the algorithm behind `grep -F` (fixed-string search) and most intrusion detection systems. But with SIMD, we can push the constant factor even lower.

## Aho-Corasick Automaton

Build a trie of all patterns, then add failure links: when a character doesn't match the current trie node, follow the failure link to the longest proper suffix that IS a prefix of some pattern. This is like KMP generalized to multiple patterns.

```rust
struct AhoCorasick {
    goto: Vec<[usize; 256]>,  // transition table
    fail: Vec<usize>,          // failure links
    output: Vec<Vec<usize>>,   // pattern IDs ending at this state
}

impl AhoCorasick {
    fn build(patterns: &[&[u8]]) -> Self {
        let mut goto = vec![[0usize; 256]]; // state 0 is root
        let mut output = vec![Vec::new()];

        // Build trie
        for (pid, pat) in patterns.iter().enumerate() {
            let mut state = 0;
            for &c in *pat {
                let c = c as usize;
                if goto[state][c] == 0 {
                    goto[state][c] = goto.len();
                    goto.push([0usize; 256]);
                    output.push(Vec::new());
                }
                state = goto[state][c];
            }
            output[state].push(pid);
        }

        // Build failure links via BFS
        let mut fail = vec![0usize; goto.len()];
        let mut queue = VecDeque::new();
        for c in 0..256 {
            if goto[0][c] != 0 {
                queue.push_back(goto[0][c]);
                fail[goto[0][c]] = 0;
            }
        }
        while let Some(r) = queue.pop_front() {
            for c in 0..256 {
                let s = goto[r][c];
                if s != 0 {
                    queue.push_back(s);
                    let mut f = fail[r];
                    while f != 0 && goto[f][c] == 0 { f = fail[f]; }
                    fail[s] = goto[f][c];
                    output[s].extend_from_slice(&output[fail[s]]);
                }
            }
        }

        AhoCorasick { goto, fail, output }
    }

    fn search(&self, text: &[u8]) -> Vec<(usize, usize)> {
        let mut matches = Vec::new();
        let mut state = 0;
        for (i, &c) in text.iter().enumerate() {
            let c = c as usize;
            while state != 0 && self.goto[state][c] == 0 {
                state = self.fail[state];
            }
            state = self.goto[state][c];
            for &pid in &self.output[state] {
                matches.push((i, pid));
            }
        }
        matches
    }
}
```

The inner loop has two unpredictable branches: the failure-link while-loop and the output check. For k = 1000 patterns, the goto table is 256K entries (1 MB) — it doesn't fit in L1 cache. The failure-link traversal is essentially random pointer chasing.

## Stage 1: Bitmap-Based Failure Transitions

The `while state != 0 && goto[state][c] == 0` loop can be eliminated by precomputing the **next state** for every (state, character) pair — the **full transition function** (deterministic finite automaton, DFA):

```rust
fn build_dfa(goto: &[[usize; 256]], fail: &[usize]) -> Vec<[usize; 256]> {
    let num_states = goto.len();
    let mut next = vec![[0usize; 256]; num_states];
    for state in 0..num_states {
        for c in 0..256 {
            let mut s = state;
            while s != 0 && goto[s][c] == 0 { s = fail[s]; }
            next[state][c] = goto[s][c];
        }
    }
    next
}
```

Now the search loop becomes:

```rust
for &c in text {
    state = next[state][c as usize];
    for &pid in &output[state] {
        matches.push((i, pid));
    }
}
```

No failure-link traversal at search time. Cost: the `next` table is `num_states × 256 × size_of(usize)` bytes. For 10,000 states, that's ~20 MB — it doesn't fit in L3. This is the classic space-time tradeoff.

## Stage 2: Compressed Transition Table with Bitmaps

Instead of a full 256-entry array per state, store a bitmap of which characters have transitions, plus a compact array of target states:

```rust
struct CompressedState {
    bitmap: [u64; 4],      // 256 bits: which characters have non-root transitions
    targets: Vec<usize>,   // compact target states, indexed by popcount of bitmap
}
```

To find `next[state][c]`:
1. If `bitmap[c] == 1`: `targets[popcount(bitmap[0..c])]`
2. Else: if `state == 0`, state 0. Otherwise, the failure link's transition (precomputed as `default`).

```rust
fn get_next(state: &CompressedState, c: u8, default: usize) -> usize {
    let word = (c / 64) as usize;
    let bit = c % 64;
    if state.bitmap[word] & (1u64 << bit) != 0 {
        let pop = (state.bitmap[word] & ((1u64 << bit) - 1)).count_ones()
                + state.bitmap[..word].iter().map(|&w| w.count_ones()).sum::<u32>();
        state.targets[pop as usize]
    } else {
        default
    }
}
```

The popcount is fast (3-cycle `popcnt` instruction). The bitmap check is branchless (conditional move). This compressed representation uses ~80 bytes per state instead of 2048 bytes — a 25× compression. The DFA for 10,000 patterns now fits in 800 KB (L2 cache on Zen 2).

## Stage 3: SIMD Multi-Character Transitions

For patterns that are all short (≤ 8 bytes), we can process 4 characters at once. Precompute transitions for all 4-byte sequences that appear in the patterns, using a hash table:

```rust
// For each state, store a hash table of 4-byte sequences → next state
struct SIMDState {
    keys: Vec<u32>,    // 4-byte sequences
    vals: Vec<usize>,  // corresponding next states
}
```

Search loop:

```rust
let mut i = 0;
while i + 4 <= text.len() {
    let chunk = u32::from_le_bytes(text[i..i+4].try_into().unwrap());
    // Look up chunk in current state's hash table
    if let Some(&next) = hash_map.get(&chunk) {
        state = next;
        i += 4;
    } else {
        // Fall back to single-character transition for text[i]
        state = next_single[state][text[i] as usize];
        i += 1;
    }
}
```

For k = 100 short patterns (virus signatures, spam keywords), this achieves ~3× speedup over the compressed DFA.

## Practical Considerations

### Double-Array Trie

The goto table can be compressed further using a double-array structure (two parallel arrays `base` and `check`). The transition for character c from state s is: if `check[base[s] + c] == s`, then next state is `base[s] + c`. This reduces memory to ~8 bytes per transition.

### When Not to Use Aho-Corasick

- **2–3 patterns**: Just run Horspool/BM for each. The automaton overhead exceeds the gain.
- **Regex patterns**: Aho-Corasick is for fixed strings. For regex, use Thompson NFA or a DFA.
- **Streaming data**: The DFA has unbounded state; the failure-link version may backtrack.

### Rabin-Karp as a Filter

For very long texts and few patterns with distinct fingerprints, use Rabin-Karp hashing as a pre-filter: only run the full comparison when the rolling hash matches one of the pattern hashes. This filters out ~99.9% of candidate positions and runs at memory bandwidth (~20 GB/s).

## Benchmark Summary (1 MB text, k patterns of length 10–20)

| k | Algorithm | Time |
|---|-----------|------|
| 2 | Horspool × 2 | 0.3 ms |
| 2 | Aho-Corasick (trie) | 0.7 ms |
| 100 | Aho-Corasick (compressed DFA) | 2.1 ms |
| 100 | Rabin-Karp filter + Horspool | 1.3 ms |
| 1000 | Aho-Corasick (compressed DFA) | 8.5 ms |
| 1000 | SIMD-accelerated AC | 3.8 ms |

The lesson: Aho-Corasick is asymptotically optimal (O(n) independent of k), but the constant factors from random memory access kill its practical performance for small k. For few patterns, run multiple single-pattern searchers. For many patterns and long texts, the compressed DFA with SIMD acceleration is the best we have.
