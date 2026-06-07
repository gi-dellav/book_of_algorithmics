# Memory-Level Parallelism

A single cache miss takes ~200 cycles (RAM). If the CPU did nothing during that time, memory bandwidth would be limited to one word per 200 cycles — about 0.04 GB/s instead of the ~20 GB/s we actually measure. The difference is **Memory-Level Parallelism** (MLP): the CPU's ability to have multiple cache misses in flight simultaneously.

## The Experiment

The pointer-chasing experiment from the latency article creates dependent loads — each load must complete before the next can start. This gives MLP = 1: only one outstanding miss at a time.

To measure MLP, create D independent pointer chains:

```rust
let mut d = 1;
while d <= 64 {
    // Create D independent random cyclic lists
    let mut indices: Vec<usize> = Vec::with_capacity(d);
    for idx in 0..d {
        indices.push(perm[idx]);  // Starting pointer for each chain
    }
    
    // Interleave access to the D chains
    for _ in 0..ITERS {
        for idx in 0..d {
            // SAFETY: indices are valid within bounds of `a`
            indices[idx] = unsafe { *a.as_ptr().add(indices[idx]) };  // Advance one step in chain d
        }
    }
    
    d *= 2;
}
```

## Results (Zen 2)

| Parallel chains (D) | Effective latency (cycles) | Bandwidth (GB/s, approx) |
|---------------------|----------------------------|--------------------------|
| 1 | ~200 | ~0.04 |
| 2 | ~100 | ~0.08 |
| 4 | ~50 | ~0.16 |
| 8 | ~30 | ~0.27 |
| 16 | ~18 | ~0.44 |
| 32 | ~14 | ~0.57 |
| 64 | ~14 | ~0.57 |

The effective latency drops as D increases — the CPU overlaps the misses. But it saturates at D ≈ 16–20: this is the maximum number of outstanding cache misses the Zen 2 memory subsystem can track.

The product of latency and bandwidth (the bandwidth-delay product) divided by the cache line size gives the MLP needed to saturate bandwidth:
```
MLP_needed = (latency × bandwidth) / line_size
           = (200 cycles × 20 GB/s) / 64 B
           = (100 ns × 20 GB/s) / 64 B
           ≈ 31
```

The measured MLP of ~18 falls short of the theoretical 31 — the gap is due to overhead in the memory controller, DRAM timing constraints (CAS latency, RAS-to-CAS delay), and the fact that multiple misses to the same DRAM bank cannot be fully overlapped.

## How the CPU Achieves MLP

1. **Out-of-Order execution**: The CPU looks ahead in the instruction stream and finds independent loads. If loop iteration i's load and iteration i+1's load are independent (common in streaming loops), the CPU dispatches both before either completes.

2. **Hardware prefetcher**: Detects patterns and issues speculative loads even before the CPU requests the data. This effectively increases MLP without consuming ROB entries.

3. **Line fill buffers (LFBs)**: Zen 2 has ~8–12 LFBs — hardware buffers that track outstanding cache misses. Each LFB holds one in-flight cache line request. When all LFBs are occupied, no more misses can be issued until one completes.

4. **Miss status holding registers (MSHRs)**: Like LFBs but for misses that hit the L2/L3 caches. Zen 2 has ~20–30 MSHRs across L1/L2.

## Implications

1. **Pointer chasing caps MLP at 1.** A linked list traversal, hash table lookup with chaining, or tree descent creates a chain of dependent loads. Each step must complete before the next begins → MLP = 1 → RAM latency limits performance.

2. **Streaming achieves high MLP.** A loop like `sum += a[i]` has no data dependency between iterations (the pointer increment is independent of the load). The OoO engine issues many loads ahead → MLP ≈ ROB_size / (instructions per iteration) ≈ 50–100. Saturates RAM bandwidth easily.

3. **Multiple accumulator loops don't increase MLP** (but they do increase ILP). The MLP is determined by how many loads can be outstanding, not by how many accumulators are used. Both single-accumulator and multi-accumulator streaming loops achieve high MLP.

4. **Software prefetching** can increase MLP for irregular access patterns by issuing loads speculatively, but it's tricky to tune correctly. See `prefetching.md`.

## Measuring MLP

```bash
perf stat -e l1d_pend_miss.pending,l1d_pend_miss.fb_full ./program
```

- `l1d_pend_miss.pending`: Number of outstanding L1 data cache misses at each cycle (average).
- `l1d_pend_miss.fb_full`: Cycles where all line fill buffers are occupied.

If `fb_full` is high, the code is limited by MLP — the CPU has enough independent work but can't issue more memory requests. The fix: reduce the number of cache misses (better locality) rather than trying to increase MLP further.
