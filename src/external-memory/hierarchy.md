# Memory Hierarchy

The memory hierarchy is not an accident — it's a direct consequence of physics and economics. Fast memory is expensive (in dollars, power, and die area). Large memory is slow (in latency). The hierarchy gives the illusion of both speed and size by keeping frequently-used data in fast levels and rarely-used data in slow levels.

## The Technologies

### Registers

- **Technology**: SRAM, integrated directly into the execution units.
- **Size**: ~200–400 registers (including renamed physical registers) per core.
- **Latency**: 0 cycles (register reads are free — they're wired into the execution units).
- **Managed by**: The compiler (register allocation) and the CPU (register renaming).

Registers are the only truly "free" storage. Every other level requires an explicit load instruction and consumes pipeline resources.

### SRAM (Static RAM): L1, L2, L3 Caches

- **Technology**: 6 transistors per bit (6T SRAM cell). Fast, power-hungry, expensive in die area.
- **L1**: 32 KB data + 32 KB instruction per core. 4–5 cycles latency. Usually 8-way set-associative.
- **L2**: 256–512 KB per core. ~12 cycles. Usually 8-way set-associative.
- **L3 (LLC)**: 4–32 MB shared across 4–8 cores. ~40 cycles. Usually 16-way set-associative.

SRAM is *static* — it retains data as long as power is applied, no refresh needed. The latency increases with size because the access path is longer and more complex (more sets → larger tag arrays → slower comparison).

### DRAM (Dynamic RAM): Main Memory

- **Technology**: 1 transistor + 1 capacitor per bit (1T1C cell). Needs periodic refresh.
- **Size**: 8–64 GB per system.
- **Latency**: ~50–100 ns (100–200 cycles at 2 GHz). Includes row activation, column access, and data transfer.
- **Bandwidth**: ~25 GB/s per channel (DDR4-3200), typically 1–2 channels.

DRAM is organized in rows and columns. Accessing a new row requires opening it (RAS — Row Address Strobe), which takes ~15 ns. Then the column can be accessed (CAS — Column Address Strobe), taking ~15 ns. Finally, data is transferred (burst of 8 words). If the next access is in the same open row, only CAS is needed — 2× faster. This is why sequential access is faster than random (the row stays open).

### Non-Volatile Storage: Flash (SSD) and Magnetic (HDD)

- **SSD (NVMe)**: ~10–100 µs latency, ~3 GB/s bandwidth, ~$0.10/GB. No moving parts. Reads are fast; writes are slower (NAND flash requires erase-before-write at block granularity).
- **HDD**: ~5–10 ms latency (seek time + rotational delay), ~200 MB/s bandwidth, ~$0.02/GB. Mechanical. Random access is devastating (~10 ms per random read = 100 reads/second).

The gap between RAM and SSD (~10 µs vs. 100 ns = 100×) is large enough that OS paging to SSD is painful. The gap between SSD and HDD (~10 µs vs. 10 ms = 1000×) is why SSDs transformed computing.

### Network Storage

- **Latency**: ~100 µs (RDMA over InfiniBand) to ~100 ms (public internet).
- **Bandwidth**: ~1 Gbps to 100 Gbps.
- **Cost**: Variable, but effectively infinite capacity.

At these latencies, the programming model must be completely asynchronous. You can't wait for a network round-trip in the middle of a computation; you must issue requests and overlap them with other work. Distributed computing (Chapter 14) covers these patterns.

## The Trends

Two graphs tell the story:

**Processor-Memory Performance Gap**: CPU speed and memory speed both grew exponentially from 1980 to 2005, but CPU speed grew faster. Since 2005, CPU single-thread performance has plateaued (Dennard scaling ended), but memory continues to improve slowly. The gap has stabilized at ~100× (100 cycles to RAM vs. 1 cycle to register), but it's not closing.

**Memory vs. Compute Energy**: A 64-bit DRAM read consumes ~100× more energy than a 64-bit integer addition. Moving data costs energy, not just time. For battery-powered devices, optimizing memory access is both a performance and a power optimization.

## Implications for Algorithms

1. **Sequential access is always faster than random.** Sequential access opens one DRAM row and reads many columns from it. Random access opens a new row on every access, paying the RAS penalty every time.
2. **Working set size determines the effective access speed.** If the working set fits in L1, everything is fast. If it spills to L2, it's slower. If it spills to RAM, it's dramatically slower. If it spills to SSD/HDD, the program is essentially stopped.
3. **Bandwidth and latency are different bottlenecks.** A streaming algorithm (sequential, predictable) is bandwidth-limited. A pointer-chasing algorithm (random, unpredictable) is latency-limited. The external memory model distinguishes these.
