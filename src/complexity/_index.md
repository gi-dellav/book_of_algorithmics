# Why Go Beyond Big O?

For fifty years, asymptotic complexity was the north star of algorithm design. Given two algorithms, you counted the operations each performed as a function of input size n, dropped the constants, and kept the fastest-growing term. O(n log n) beats O(n²). Full stop. This was not wrong — it was correct *for the hardware it was designed on*.

## The RAM Model

Classical algorithm analysis uses the **Random Access Machine** (RAM) model. In this model:

1. Each simple operation (addition, multiplication, comparison, memory access) costs **one unit of time**.
2. Memory is **flat** — accessing any address costs the same as any other.
3. The machine is **sequential** — one operation happens at a time.

Under these assumptions, counting operations gives you a faithful proxy for running time. An algorithm that performs 2n log n operations is twice as fast as one that performs 4n log n operations, regardless of what those operations *are*.

The RAM model was designed in the 1960s, when computers looked like the IBM System/360. Memory access took roughly as long as arithmetic. There was no cache, no pipeline, no branch predictor. An instruction was fetched, decoded, executed, and retired — one at a time, in order. Counting instructions counted time.

## What Changed

Between 1970 and 2020, CPU performance improved by roughly a factor of 10⁶. Memory performance improved by roughly a factor of 10³. The gap between processor speed and memory speed widened by three orders of magnitude.

To bridge this gap, architects added caches — small, fast memories that sit between the CPU and main memory. The L1 cache on a modern processor can service a request in 4–5 cycles. Main memory takes 50–100 cycles. An SSD takes 10⁵ cycles. A spinning disk takes 10⁷ cycles.

The RAM model's flat memory assumption is now wrong by a factor of 10⁷.

Architects also added instruction-level parallelism. A modern CPU core can execute 4–6 instructions *simultaneously*, reordering them on the fly to keep its execution units busy. Whether your code achieves this depends on dependency chains, branch predictability, and register pressure — none of which appear in asymptotic analysis.

And then there are SIMD instructions, which operate on 4, 8, or 16 values at once. A vectorized loop can be 8× faster than a scalar loop doing the same number of "operations." The RAM model doesn't distinguish between a scalar add and a vector add.

## The Cost Hierarchy

Here is a rough guide to the cost of operations on a modern Zen 2 core (Ryzen 7 4700U, 2.0 GHz base clock):

| Operation | Latency | Throughput (per cycle) |
|-----------|---------|------------------------|
| Integer addition | 1 cycle | 4 |
| Integer multiplication | 3 cycles | 1 |
| Float addition (scalar) | 3 cycles | 1 |
| Float multiplication (scalar) | 3 cycles | 1 |
| Float division (scalar) | 13–15 cycles | 1/13 |
| L1 cache hit | 4–5 cycles | 2 loads |
| L2 cache hit | 12 cycles | 1 |
| L3 cache hit | 40 cycles | ~1/3 |
| RAM access | 50–100 cycles | ~1/20 |
| Branch mispredict | 15–20 cycles | — |
| `_mm256_add_ps` (8 floats) | 3 cycles | 2 |

The takeaway: a single cache miss costs as much as 100 additions. A branch mispredict costs as much as 15–20 additions. An AVX2 vector addition processes 8 floats for the same cost as a single scalar addition.

If you are counting operations but ignoring *which* operations, your estimate can be wrong by a factor of 100 before you even consider constant factors.

## When Big O Still Matters

None of this means asymptotic complexity is useless. It is the right first cut. An O(n²) algorithm will always lose to an O(n log n) algorithm for sufficiently large n. The question is: *what is "sufficiently large"?*

For sorting, n ≈ 50 is where quicksort (O(n log n)) starts beating insertion sort (O(n²)) on real hardware. But for other problems, the crossover can be enormous. There are problems where the "asymptotically slower" algorithm wins for every n that fits in memory, because the constant factor from cache effects overwhelms the asymptotic advantage until n exceeds 10⁹.

Big O tells you how an algorithm scales. It does not tell you how fast it is. For that, you need to understand the machine.
