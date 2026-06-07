# Chapter: Complexity Models (`complexity/`)

## Overview

Weight 1 in the book — the opening chapter. It sets the philosophical and historical stage for why asymptotic complexity (Big O) is no longer sufficient for predicting real-world performance. It covers the history of computing from 1960s supercomputers through microchips, Dennard scaling, Moore's law, and the shift to parallel architectures. It then demonstrates the staggering performance variance across programming languages using matrix multiplication as a case study. The `_index.md` explains that classical complexity theory counted instructions, but modern hardware makes this off by "orders of magnitude."

## Files and Content

| File | Status | Size | Description |
|------|--------|------|-------------|
| `_index.md` | Complete | 4.9 KB | Introduction to computational complexity, instruction latencies, asymptotic complexity, and why Big O was once sufficient but no longer is |
| `hardware.md` | Complete | 10.7 KB | History: microchips, photolithography, Dennard scaling, Moore's law, the 2005 leakage wall, modern approaches (pipelining, OoO, superscalar, SIMD, caching, multicore, GPU, FPGA/ASIC). Die shot of Zen CPU core. |
| `languages.md` | Published | 12.2 KB | Matrix multiplication in Python (630s), Java (10s), JIT Python/PyPy (12s), C with `-O3` (9s), C with `-march=native -ffast-math` (0.6s), BLAS/numpy (0.12s). The lesson: native languages give *control*, not automatic performance. |
| `levels.md` | Draft | 7.1 KB | Rambling but insightful: 6 "levels" of optimization expertise (newbie → Intel employee), when to optimize, latency vs. efficiency tradeoffs in web services, and career path thoughts |
| `models.md` | Draft | 1.9 KB | Brief sketch of computation models beyond RAM: word RAM, external memory, parallel RAM, communication complexity, quantum, energy models |

## Image Assets

7 images: `complexity.jpg` (93.3 KB — asymptotic complexity cartoon), `cpu.png` (41.9 KB — CPU diagram), `dennard.ppm` (77.6 KB — Dennard scaling data), `die-shot.jpg` (2.2 MB — Zen CPU die photo), `lithography.png` (85.6 KB — photolithography process), `mos-6502.jpg` (155.0 KB — historic microprocessor), `bug.jpg` (89.5 KB).

## Strengths

1. **Compelling narrative arc**: The chapter tells a coherent story: complexity theory was right *for its era* → hardware changed dramatically → we need new models. The historical photos and graphs make this tangible.
2. **`languages.md` is a powerful demo**: The 5250x speedup from plain Python to BLAS (630s → 0.12s) on identical hardware is the single most effective attention-grabber in the book. It proves the book's thesis in one concrete example.
3. **Beautiful hardware explanation**: The photolithography explanation, Dennard scaling derivation, and leakage wall explanation are clear and accessible without assuming an EE background.
4. **The Zen die shot**: The 1.4-billion-transistor die photo with the caption contextualizes the scale of modern hardware.
5. **Honest about limitations**: The chapter doesn't claim asymptotic complexity is useless — just that it's insufficient. The nuanced position is well-articulated.

## Areas for Improvement

1. **`levels.md` is chaotic**: It jumps from Google hiring practices to pageview economics to factorization to SIMDJSON to latency vs. efficiency. While the ideas are interesting, the article needs a clear structure.
2. **`models.md` is a stub**: At 1.9 KB, it only lists model names without explaining any of them. The external memory model is covered in a later chapter, but parallel RAM, communication complexity, and energy models are never developed.
3. **No quantitative bridge**: The chapter doesn't provide concrete "rules of thumb" for how far off Big O predictions can be (e.g., "An O(n log n) algorithm that is cache-friendly can beat an O(n) algorithm that pointer-chases, up to n ≈ 10⁹").
4. **Language comparison is incomplete**: Only Python, Java, and C are compared. Missing: Go, Rust, Julia, JavaScript (Node/V8) — which could strengthen the "language doesn't determine performance" argument.
5. **The `languages.md` code listings are verbose**: The full matrix multiplication code in each language takes up significant space. Could be condensed with a table showing key differences.
6. **No forward roadmap**: The chapter doesn't preview what's coming in the rest of the book or how the remaining chapters address the problems it identifies.

## Recommendations

1. **Restructure `levels.md`**: Break into (a) "When to optimize" — latency vs. throughput tradeoffs, (b) "The levels of optimization expertise" — the 0-5 scale, (c) "Estimating impact" — whether the optimization matters in context.
2. **Expand `models.md`**: Give at least one concrete example per model (e.g., external memory model: B-tree I/O complexity; parallel RAM: Brent's theorem). Even 1-2 pages per model would make this a useful reference.
3. **Add quantitative heuristics**: Create a "cheat sheet" like: "L1 cache hit ~4 cycles, RAM access ~100 cycles, SSD ~10⁵ cycles. An algorithm that does N random RAM accesses is bounded by ~100N ns, regardless of Big O."
4. **Add a "What's ahead" section**: End the chapter with a brief preview of each subsequent chapter and which aspect of hardware it addresses (architecture → what the CPU does, pipelining → how it overlaps work, SIMD → data parallelism, cache → memory hierarchy, etc.).
5. **Add Rust/Go benchmarks**: Show that idiomatic Rust with iterators can match C performance, while naive Go can be slower than Java — reinforcing the "control, not language" thesis.
6. **Provide downloadable benchmark code**: The matrix multiplication comparison would make an excellent hands-on exercise. Link to a repo where readers can reproduce the results on their own hardware.
