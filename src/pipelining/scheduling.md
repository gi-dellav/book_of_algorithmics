# Scheduling and µops

The CPU doesn't execute x86 instructions directly. It translates them into a simpler internal representation — **µops** (micro-operations) — and schedules those µops across its execution units. Understanding this translation and scheduling explains many otherwise mysterious performance effects.

## Instruction Translation

When an x86 instruction enters the decoder, it's cracked into one or more µops:

```asm
add rax, [rbx]       ; → 1 µop: load-and-add (fused)
add [rbx], rax       ; → 2 µops: load, add-and-store
rep movsb            ; → many µops (implemented by microcode)
idiv rcx             ; → many µops (microcoded division)
```

Simple ALU operations with register operands become a single µop. Operations that touch memory or combine arithmetic with memory access may be split. Complex instructions (string ops, division, transcendental functions) are handled by **microcode** — a ROM that emits a sequence of µops.

**µop fusion**: Certain patterns (like `cmp`+`jcc`, or load+ALU) are "fused" into a single µop. This saves issue bandwidth and ROB space. Zen 2 can fuse:
- Branch fusion: `cmp`/`test` + conditional jump → 1 µop.
- Load-op fusion: `add rax, [mem]` → 1 µop.

The fused µop is later split at execution time (the load and add target different execution units), but it only consumes one ROB entry and one issue slot.

## The Execution Pipeline (Zen 2 specific)

After decoding, µops flow through:

1. **µop Queue**: Buffers decoded µops, decoupling front-end from back-end.
2. **Reorder Buffer (ROB)**: 224 entries. Tracks in-flight µops in program order.
3. **Scheduler (Reservation Stations)**: 4× 24-entry queues (one per integer pipe, one shared for FP). Holds µops waiting for operands.
4. **Execution Units**:
   - **Integer**: 4 ALU pipes (ports 0–3). All can do basic integer ops. Ports 0 and 1 also handle branches. Port 2 handles `lea` and multiply. Port 3 handles `lea` only.
   - **Load/Store**: 2 load pipes (ports 2, 3) + 1 store pipe (port 7). Can execute 2 loads + 1 store per cycle.
   - **Floating-Point**: 4 pipes (ports 0–3). Port 0: FMA, FP add/mul. Port 1: FMA, FP add/mul. Port 2: FP add. Port 3: FP add, FP-to-int conversion.
5. **Retirement**: Up to 8 µops/cycle are retired in program order, committing results to the architectural state.

## The Scheduler and Out-of-Order Dispatch

The scheduler is the heart of OoO execution. Each cycle, it scans all waiting µops and finds those whose operands are ready. It assigns them to free execution units.

The scheduler can see 4 × 24 = 96 µops on Zen 2. If none of those µops have ready operands (because they're all waiting on a cache miss, for example), the execution units sit idle — even though there are instructions in the ROB. This is why **instruction-level parallelism** requires having many independent operations in the visible instruction window.

The ROB's 224 entries mean the CPU can look ahead up to ~224 µops (roughly 100–150 x86 instructions) searching for independent work. Long dependency chains or frequent cache misses that fill the ROB with waiting instructions reduce the effective window.

## Port Pressure

Each execution unit is accessible through specific **ports**. If a loop uses only instructions that go through port 0, port 0 is at 100% utilization while ports 1–3 are idle — you get at most 1 µop/cycle throughput.

Analyzing port pressure:

```bash
llvm-mca -march=znver2 your_loop.s
```

Output shows:
```
Resource pressure by instruction:
[0]    [1]    [2]    [3]    [4]    [5]    [6]    [7]
2.00    -     2.00   1.00    -      -     1.00    -
```
Where [0]–[7] correspond to execution ports. If any port exceeds the number of iterations, that port bottlenecks the loop.

Common port-bottleneck scenarios:
- **Load-heavy code**: Ports 2 and 3 (max 2 loads/cycle). If your loop loads 4 values per 3-element computation, loads are the bottleneck.
- **Store-heavy code**: Port 7 (max 1 store/cycle). Store bandwidth is half of load bandwidth on Zen 2.
- **Integer multiply**: Port 2 only (throughput 1/cycle). A loop doing `imul` on every element caps at 1/cycle before considering other instructions.
- **FP-to-int conversion**: Port 3 only (throughput 1/cycle).

## Scheduling Examples

### Independent Integer Adds

```asm
.Lloop:
    add eax, [rdi]      ; load (port 2/3) + add (port 0/1/2/3) → fused to 1 µop
    add eax, [rdi+4]
    add eax, [rdi+8]
    add eax, [rdi+16]
    add rdi, 32
    cmp rdi, rsi
    jne .Lloop
```

But these adds are all to `eax` — they form a dependency chain. The scheduler serializes them. Throughput: 1 add per 4 cycles (load latency) = 0.25 IPC.

### Same Adds, Multiple Accumulators

```asm
.Lloop:
    add eax, [rdi]       ; accumulator 0
    add ebx, [rdi+4]     ; accumulator 1 (independent)
    add ecx, [rdi+8]     ; accumulator 2 (independent)
    add edx, [rdi+16]    ; accumulator 3 (independent)
    add rdi, 32
    cmp rdi, rsi
    jne .Lloop
```

Now the four `add` µops are independent. The scheduler can issue all four simultaneously (if enough load pipes and ALU pipes are available). Throughput approaches 4 adds per 4 cycles × 1 cycle = ~4 IPC.

## Why You Should Care

1. **`llvm-mca` is your friend**: Before hand-optimizing assembly, run it through llvm-mca to see the predicted throughput and bottleneck. It's often not what you expect.

2. **µop fusion is invisible but beneficial**: The compiler works hard to generate fusable patterns. Don't add unnecessary instructions between a `cmp` and its `jcc`.

3. **Port pressure is a real limit**: Even if your code has no dependency chains and no cache misses, you can't exceed the issue width of the execution ports you're using. SIMD often shifts the bottleneck from ports to memory bandwidth.

4. **The ROB is finite**: Avoid extremely long dependency chains even across loop iterations. The CPU can't look past the end of the ROB — if the ROB fills with stalled instructions, it can't fetch new ones that might be independent.
