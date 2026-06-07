# Applied Algorithm Optimization

The preceding chapters gave you the tools. This chapter applies them. Each case study takes a real algorithm — something you might actually implement — and pushes it from a naive baseline to a highly-optimized implementation on real hardware.

## What Makes These Case Studies Different

These are not toy examples. Each one:
- Starts with a working, correct implementation.
- Identifies the bottleneck (cache, branches, dependency chains, SIMD width).
- Applies techniques from the relevant chapters.
- Measures the speedup at each step.
- Reports concrete performance numbers on Zen 2.

The progression matters. In matrix multiplication, we go from 0.2 GFLOPS (naive triple loop) to 28 GFLOPS (blocked + SIMD + register reuse) — a 140× speedup. Each step is explained: why this change, what the bottleneck is now, what the next step targets.

## The Algorithms

1. **Integer Factorization** — Trial division → wheel factorization → Pollard's rho → Pollard-Brent. The lesson: algorithmic improvement matters more than micro-optimization, until it doesn't.
2. **Binary GCD** — Euclid → binary GCD → Lemire-Corderoy optimization. ~2× faster than `std::gcd`.
3. **Argmin with SIMD** — Scalar → vector of indices → branch vs. comparison-then-index. ~15× faster.
4. **Prefix Sum** — Scalar → SIMD with blocking and continuous loads. ~2.5× faster.
5. **Matrix Multiplication** — The crown jewel. 7 stages from naive to BLAS-level in <40 lines of C. ~140× speedup.
6. **Logistic Regression** — Quantization, SIMD dot product, removing softmax.
7. **Integer Parsing** — SWAR techniques and AVX-512 for parsing integers from strings.
8. **Sorting** — Quicksort optimization, radix sort, SIMD sorting networks.

## Recommended Reading Order

If you read just one case study, make it **Matrix Multiplication**. It ties together cache blocking (Chapter 8), SIMD (Chapter 10), instruction-level parallelism (Chapter 3), and benchmarking (Chapter 5).

If you want to see the biggest speedup from algorithmic insight alone, read **Factorization**. The progression from trial division to Pollard-Brent is a masterclass in algorithmic thinking.

If you want to see how SIMD transforms a fundamentally scalar problem, read **Argmin** and **Prefix Sum**.

The data structures chapter that follows applies these same techniques to search trees, hash tables, and segment trees — completing the journey from hardware to algorithms to data structures.
