# Pipelining, Superscalar, Out-of-Order

A modern CPU core executes instructions nothing like the sequential, one-at-a-time model you learned in computer organization class. Understanding the reality — pipelining, superscalar execution, and out-of-order processing — is the key to writing code that keeps the machine busy.

## An Education Analogy

Imagine a school with four classrooms. Each student must pass through four classes in sequence: Math, Physics, Chemistry, and Biology. Each class takes one hour.

**Sequential model**: One student at a time completes all four classes. The school graduates one student every 4 hours. Throughput: 0.25 students/hour.

**Pipelined model**: As soon as Student 1 finishes Math and moves to Physics, Student 2 starts Math. At steady state, all four classrooms are busy simultaneously, and a student graduates every hour. Throughput: 1 student/hour — 4× better, with the same classrooms and teachers.

This is **pipelining**: overlapping the execution of multiple instructions by splitting the work into stages. The latency for any one student is still 4 hours, but the throughput is 4× higher.

**Superscalar model**: We build a second set of classrooms. Two students can take Math simultaneously (if two math teachers are available). Students can even skip ahead if they don't need a particular class. Throughput: up to 2 students/hour.

This is **superscalar execution**: the CPU has multiple execution units of each type and can dispatch multiple independent instructions in the same cycle.

**Out-of-Order model**: Some students finish Physics faster than others. Rather than making a fast student wait because the student ahead of them is slow in Chemistry, we let students proceed to whichever classroom is free, as long as prerequisites are satisfied. A fast student might graduate in 2 hours; a slow one in 6. Throughput increases because classrooms are rarely idle.

This is **out-of-order execution**: instructions execute as soon as their operands are available, not in program order. The CPU maintains the illusion of sequential execution (retiring instructions in order), but internally they run in whatever order minimizes idle time.

## Latency vs. Throughput

A critical distinction:

- **Latency**: The time from issuing one instruction to its result being available. Determines how fast a dependency chain executes.
- **Throughput**: The rate at which the CPU can start new independent instructions of the same type. Determines how fast a loop with no data dependencies executes.

Example: Zen 2 floating-point addition has latency 3 cycles and throughput 1 per cycle (actually 2 per cycle with both FMA pipes). This means:
- `x = a + b; y = x + c;` → `y` is available 6 cycles after `a` and `b` (two serial adds, each 3 cycles).
- `x = a + b; y = c + d;` → both results are available after 3 cycles (the two adds execute in parallel on different pipes).

The **critical path** is the longest chain of dependent operations. Its length, multiplied by the latency of each operation, sets a lower bound on execution time.

## The Classic Five-Stage Pipeline

The simplest pipeline (as taught in textbooks, and roughly what a 1990s RISC CPU used):

1. **IF** (Instruction Fetch): Load instruction from I-cache.
2. **ID** (Instruction Decode): Decode opcode, read registers.
3. **EX** (Execute): Perform ALU operation, compute memory address.
4. **MEM** (Memory Access): Load from or store to data cache.
5. **WB** (Write Back): Write result to register file.

Each stage takes one cycle. An instruction enters the pipeline and advances one stage per cycle. At steady state, one instruction completes per cycle (if no stalls).

Real x86 CPUs have 14–19 stages. The extra stages handle: x86 instruction decoding (complex, variable-length), µop translation, µop caching, address generation, multiple levels of data cache access, and result forwarding.

## Superscalar Execution

A superscalar CPU issues multiple instructions per cycle. Zen 2 can:
- **Fetch** 32 bytes (roughly 8 instructions) per cycle.
- **Decode** 4 x86 instructions per cycle.
- **Dispatch** up to 6 µops per cycle to the execution units.
- **Retire** up to 8 µops per cycle.

The execution units (simplified Zen 2 layout):
- 4 integer ALUs
- 3 AGUs (address generation, for load/store)
- 2 load pipes + 1 store pipe (can do 2 loads + 1 store per cycle)
- 4 FP/vector pipes (can do 2× FMA + 2× FMA, or various mixes of float/SIMD ops)

To achieve 6 µops/cycle, the CPU must find 6 µops whose inputs are ready and whose required execution units are free. This is the job of the scheduler / instruction scheduler.

## Out-of-Order Execution: The Reorder Buffer

Instructions enter the pipeline in program order (the **front-end** is in-order). After decoding, they're placed in the **reorder buffer** (ROB) — a circular queue that tracks up to ~224 in-flight µops on Zen 2.

From the ROB, µops are dispatched to the **scheduler** (a.k.a. reservation stations), which holds µops waiting for their operands. When all operands are ready and an appropriate execution unit is free, the µop issues to that unit. After execution, the result is broadcast to all waiting µops (bypassing the register file — this is called **forwarding** or **bypassing**).

µops complete execution out of order. But they **retire** (commit their results to the architectural state) in program order. The ROB ensures that: (a) the architectural state (registers, memory) never reflects an execution order different from the program order, and (b) exceptions are precise — if instruction N faults, instructions N+1, N+2, ... haven't committed yet, so their effects can be discarded.

## Why This Matters for Your Code

1. **Dependency chains limit ILP**: If each instruction depends on the previous one, the CPU can't parallelize them. You get 1 instruction per latency cycles. Write code to minimize long chains.

2. **Independent operations are free (in terms of latency)**: `x = a+b; y = c+d;` are two independent adds. They execute in parallel (on different pipes) and both complete in 3 cycles.

3. **Throughput, not latency, determines loop performance**: If each loop iteration is independent, you can start a new iteration every cycle (limited by throughput), even if each iteration takes 5 cycles to complete. The pipeline overlaps them.

4. **The ROB size limits the instruction window**: ~224 µops on Zen 2. If your loop body is 100 µops, the CPU can look ahead about 2 iterations to find independent work. If the body is 5 µops, it can look ahead ~45 iterations. Short, simple loop bodies give the OoO engine more to work with.

5. **The µop cache matters**: The ROB + scheduler form the OoO back-end. The µop cache feeds them. A loop that fits in the µop cache (~4096 µops) bypasses the decode stage entirely, saving power and improving throughput.

The following articles explore each of these effects in detail: pipeline hazards, branch prediction, branchless coding, instruction tables, throughput optimization, scheduling, and theoretical limits.
