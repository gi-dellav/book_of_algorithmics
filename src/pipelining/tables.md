# Instruction Tables

To optimize at the instruction level, you need to know two numbers for each instruction: **latency** and **throughput**. These numbers vary by microarchitecture, but the Zen 2 (Ryzen 3000/4000 series) values we use throughout the book are representative of modern high-performance CPUs.

## Definitions

**Latency**: The number of cycles from when the instruction's inputs are ready to when its result is available to a dependent instruction. This includes forwarding — the result can be used by a µop in the very next cycle after execution completes.

**Throughput (Reciprocal Throughput)**: The number of cycles the execution unit is occupied. If an instruction has throughput 0.5, the CPU can start two such instructions per cycle (on different execution units). If it has throughput 3, the CPU can start one every 3 cycles.

The distinction: a floating-point multiplication has latency 3 and throughput 0.5 on Zen 2. You can start 2 multiplies per cycle, but each takes 3 cycles to complete. A chain of dependent multiplies runs at one per 3 cycles. Independent multiplies run at 2 per cycle.

## Zen 2 Instruction Latencies (Selected)

| Instruction | Operands | Latency | Throughput |
|-------------|----------|---------|------------|
| `add`, `sub`, `cmp`, `and`, `or`, `xor` | reg, reg | 1 | 0.25 |
| `add`, `sub` | reg, imm | 1 | 0.25 |
| `imul` | reg, reg (64-bit) | 3 | 1 |
| `imul` | reg, imm (64-bit) | 3 | 1 |
| `mul` (unsigned) | reg (full 128-bit) | 3 | 1 |
| `div` (unsigned) | reg (64-bit) | 17–44 | 17–44 |
| `idiv` (signed) | reg (64-bit) | 17–44 | 17–44 |
| `lea` | [simple] | 1 | 0.25 |
| `lea` | [complex, 3 components] | 3 | 1 |
| `shl`, `shr`, `sar` | reg, cl (variable) | 1 | 0.5 |
| `mov` | reg, reg | 0 (eliminated) | — |
| `mov` | reg, [mem] (L1 hit) | 4 | 0.5 (2 loads/cycle) |
| `mov` | [mem], reg (L1 hit) | 1 (no data dependency on store) | 1 (1 store/cycle) |
| `fadd` (scalar) | xmm, xmm | 3 | 0.5 |
| `fmul` (scalar) | xmm, xmm | 3 | 0.5 |
| `fma` (scalar) | xmm, xmm, xmm | 4 | 0.5 |
| `fdiv` (scalar, single) | xmm, xmm | 13 | 13 |
| `fsqrt` (scalar, single) | xmm, xmm | 20 | 20 |
| `vaddps` (packed, AVX2) | ymm, ymm, ymm | 3 | 0.5 |
| `vmulps` (packed, AVX2) | ymm, ymm, ymm | 3 | 0.5 |
| `vfma...ps` (packed, AVX2) | ymm, ymm, ymm | 4 | 0.5 |
| `_popcnt` | reg, reg | 1 | 0.25 |
| `lzcnt`, `tzcnt` | reg, reg | 1 | 0.25 |
| `bsr`, `bsf` | reg, reg | 3 | 1 |
| `cmovcc` | reg, reg | 1 | 0.5 |

Key insights:
- **Integer add/sub/logic is practically free**: 4 per cycle, latency 1.
- **Integer multiply is 3× slower than add**: Plan accordingly.
- **Division is devastating**: 17–44 cycles, not pipelined. Avoid in hot code.
- **Float add/mul/FMA have the same latency as integer multiply (3 cycles)**: FMA does a multiply *and* an add in 4 cycles — effectively 2 operations for the price of 1.3.
- **`mov reg, reg` is eliminated**: The CPU never executes it; it's resolved during register renaming. Zero-cost copies.
- **Load latency includes L1 cache access**: 4 cycles for L1 hit; much more for misses.

## The FMA Advantage

A fused multiply-add does `d = a * b + c` with a single rounding step. This is both faster (4 cycles vs. 3+3=6 for separate multiply and add) and more accurate (one rounding error instead of two).

Zen 2 has two 256-bit FMA units. Each can do 8 single-precision FMAs per cycle:
- Peak FLOPS = 2 units × 8 floats × 2 ops (mul+add) = 32 single-precision GFLOPS at 2 GHz.

Using FMA wherever possible is free performance. The compiler handles this when `-ffast-math` is enabled (FMA changes rounding behavior, so it's not strictly IEEE-754 compliant).

## Variable-Latency Instructions

Some instructions have data-dependent latency:

- **Division**: The latency depends on the number of significant bits in the operands. Dividing small numbers is faster.
- **Floating-point denormal handling**: Operations on denormalized numbers are much slower (microcode assist, ~100+ cycles). Flush denormals to zero with `-ffast-math` or `FTZ`/`DAZ` control flags.
- **Rep-prefixed string operations** (`rep movsb`, `rep stosb`): The "fast string" implementation in microcode can achieve high throughput for large counts, but small counts have significant overhead.

## Locating Instruction Data

For a CPU not listed here, check:
- **[Agner Fog's instruction tables](https://agner.org/optimize/)**: The gold standard. Covers x86 microarchitectures from Pentium to Zen 4.
- **[uops.info](https://uops.info/)**: Automated measurements with a web interface.
- **CPU vendor optimization manuals**: Intel's Optimization Reference Manual and AMD's Software Optimization Guide.
- **[llvm-mca](https://llvm.org/docs/CommandGuide/llvm-mca.html)**: Simulates instruction scheduling on a specific microarchitecture and reports expected throughput.

## Using Instruction Tables

When analyzing a loop:

1. List all instructions in the critical path (the longest dependency chain).
2. Sum their latencies → lower bound in cycles.
3. For each execution port, sum the µop counts → if any port exceeds the number of cycles from step 2, that port is the bottleneck.
4. The minimum time is max(critical_path, port_bottleneck).

Example: a loop with 100 independent integer adds and no dependencies. Critical path = 0 (no dependencies). Each add uses an integer ALU port. Zen 2 has 4 ALU pipes with throughput 0.25 each → 4 adds/cycle. 100 adds / 4 per cycle = 25 cycles minimum. (Plus loop overhead, loads, etc.)

We'll formalize this analysis in the `limits.md` article.
