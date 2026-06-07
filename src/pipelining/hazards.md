# Pipeline Hazards

A **hazard** is anything that prevents the next instruction from executing in the next clock cycle. There are three classes: structural, data, and control hazards. Understanding them tells you *what* can slow down your code at the hardware level.

## Structural Hazards

A structural hazard occurs when two instructions need the same hardware resource at the same time.

**Example**: A processor with a single memory port. An instruction fetch and a data load can't happen simultaneously. One must stall.

**Example**: A processor with one integer multiplier. Two multiply instructions in consecutive cycles — the second waits until the first finishes.

Modern high-performance CPUs solve structural hazards by **duplicating resources**: separate I-cache and D-cache (Harvard architecture), multiple execution units of each type, multiple load/store pipes. Structural hazards are rare in CPUs designed after ~2000. They remain relevant for specialized units (division, cryptography, AES-NI) — if you issue two AES instructions back-to-back and there's only one AES unit, the second stalls.

## Data Hazards

A data hazard occurs when an instruction depends on the result of a previous instruction that hasn't completed yet.

**Read After Write (RAW)** — the true dependency:

```asm
add rax, rbx     ; writes rax
sub rcx, rax     ; reads rax — must wait for add to complete
```

The `sub` cannot execute until `add` produces `rax`. This is a fundamental limit — no amount of hardware cleverness can eliminate a true data dependency.

**Write After Read (WAR)** and **Write After Write (WAW)** are "false" dependencies caused by register reuse. Modern CPUs eliminate them via **register renaming**: each write to a register allocates a new physical register from a pool of ~100+. Instructions that write the same architectural register actually write different physical registers, removing the false dependency.

**Forwarding (Bypassing)** mitigates RAW hazards: the result from the ALU is forwarded directly to the next instruction's input, skipping the register file write-read round trip. This reduces the effective latency — instead of waiting for WB (write-back) to finish, the dependent instruction can start as soon as the result leaves the ALU.

```asm
add rax, rbx     ; result available at end of EX stage
sub rcx, rax     ; gets rax via forwarding — only 1 cycle stall, not 2
```

Despite forwarding, chains of dependent operations are limited by the **latency** of each operation. `x = a+b; y = x+c; z = y+d;` takes 3 × 3 = 9 cycles on Zen 2 (float add latency 3), even with perfect forwarding.

## Control Hazards

A control hazard occurs when the CPU doesn't know which instruction to fetch next.

**Unconditional jumps**: The target is known at decode time. The CPU can fetch the target immediately, causing a 1–2 cycle bubble (the instructions already fetched after the jump are wrong).

**Conditional branches**: The target depends on a comparison whose result isn't known until execute time (or later). The CPU **predicts** the outcome and starts fetching down the predicted path. If the prediction is wrong, all instructions fetched and partially executed on the wrong path must be discarded (pipeline flush), and the correct path is fetched from scratch.

The misprediction penalty: 15–20 cycles on Zen 2. This is the time from when the branch executes (and the misprediction is detected) to when the correct path's instructions begin executing. During this window, the pipeline fills with useless work.

**Indirect branches** (`jmp rax`, `call [rax]`): The target must be computed or loaded from memory. The indirect branch predictor guesses the target. If wrong, the same ~20 cycle penalty applies, plus the time to compute the correct target.

## Pipeline Bubbles and Stalls

When the CPU can't issue an instruction for any reason, it inserts a **bubble** — a no-operation cycle that propagates through the pipeline. Bubbles reduce IPC (instructions per cycle).

Causes of bubbles:
- Data hazards (waiting for a result).
- Cache misses (waiting for memory).
- Branch mispredictions (waiting for the correct path to refill).
- Resource contention (structural hazard, rare).
- Instruction decode limitations (complex instructions may decode to many µops, stalling the decoder).

The impact on performance is straightforward: if your code averages 2 instructions per cycle (IPC = 2) on a machine capable of IPC = 4, then roughly half the pipeline slots are bubbles. Profiling hardware counters (`perf stat -e uops_issued.any,uops_retired.retire_slots`) tells you how many issue slots are wasted.

## Hazard Resolution Strategies

| Hazard | Hardware Solution | Software Implication |
|--------|-------------------|---------------------|
| RAW (true data dependency) | Forwarding, OoO execution | Minimize dependency chain length; break chains with multiple accumulators |
| WAR / WAW (false dependency) | Register renaming | None — hardware handles this transparently |
| Control (branch) | Branch prediction | Make branches predictable; eliminate branches with `cmov` or masking |
| Structural (resource) | Duplicate execution units | Be aware of low-throughput operations (division, random number generation) |

The hardware handles most hazards automatically. The ones you can control — data dependency chains, branch predictability, and resource contention — are the ones that repay attention in your code.
