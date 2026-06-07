# The Cost of Data Movement

How long does it take to add two numbers?

If both numbers are in registers: 1 cycle. If one is in L1 cache: 4–5 cycles. If one is in L2: 12 cycles. In L3: 40 cycles. In RAM: 100 cycles. On an SSD: 100,000 cycles. On a spinning disk: 10,000,000 cycles.

The gap between the fastest and slowest "add two numbers" operations is a factor of 10,000,000. The numbers being added are the same width. The addition circuit is the same. The difference is entirely in where the numbers are located — the cost of data movement.

This chapter develops the theoretical framework for understanding memory-bound computation. It introduces the external memory model, surveys the memory hierarchy, analyzes cache policies, presents cache-oblivious algorithms, and covers practical topics like external sorting, virtual memory, and sublinear algorithms.

## Why a Separate Chapter?

Chapters 8 and 9 are two sides of the same coin:
- **Chapter 8** (this one): The theory — models, policies, asymptotic analysis, algorithm design principles.
- **Chapter 9** (`cpu-cache/`): The experiment — measuring real caches, observing effects, building intuition.

You can read either first. The theory chapter explains *why* the experimental results look the way they do. The experimental chapter gives concrete numbers and demonstrates that the theory actually works.

## The Memory Hierarchy in One Table

| Level | Size | Latency | Bandwidth | Cost/GB |
|-------|------|---------|-----------|---------|
| Register | ~1 KB | 0 cycles | ~1 TB/s | "Free" with CPU |
| L1 cache | 32 KB | 4–5 cycles | ~1 TB/s | Included |
| L2 cache | 512 KB | 12 cycles | ~500 GB/s | Included |
| L3 cache | 8 MB | 40 cycles | ~200 GB/s | Included |
| RAM (DDR4) | 16 GB | 50–100 ns | ~25 GB/s | ~$5 |
| SSD (NVMe) | 1 TB | 10–100 µs | ~3 GB/s | ~$0.10 |
| HDD | 10 TB | 5–10 ms | ~200 MB/s | ~$0.02 |

These numbers shift every year, but the ratios remain roughly constant: each level is ~10–100× slower and ~10–100× larger than the one above it. An algorithm that touches RAM at random (pointer chasing through a large data structure) pays the RAM latency on every access. At 100 ns per access, that's 10 million accesses per second — and a modern CPU can do 2 billion operations in that same second.

The art of memory-bound algorithm design is converting random accesses into sequential scans, and sequential scans into cache-resident operations.
