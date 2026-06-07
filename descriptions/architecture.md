# Chapter: Computer Architecture (`architecture/`)

## Overview

Weight 2 in the book. This chapter teaches the fundamentals of how CPUs work, starting from assembly language and building up through functions, indirect branching, machine code layout, and concluding with loops and conditionals. The `_index.md` frames learning architecture as a prerequisite to avoid the mistake of "blindly hoping for improvement" through trial-and-error optimization.

## Files and Content

| File | Status | Size | Description |
|------|--------|------|-------------|
| `_index.md` | Complete | 1.1 KB | Motivational introduction: learn architecture before optimizing |
| `assembly.md` | Published | 10.6 KB | Assembly language tutorial. Covers Arm vs. x86 syntax, registers, mov, addressing modes, lea trick, Intel vs. AT&T syntax. |
| `functions.md` | Published | 10.7 KB | Functions, stack, calling conventions, inlining, tail call elimination. |
| `indirect.md` | Complete | 4.6 KB | Indirect branching: computed jumps, switch→branch tables, virtual method tables (vtable), dynamic dispatch overhead. |
| `interaction.md` | Draft | 747 B | Very brief sketch of interrupts and syscalls. Only shows a `write`+`exit` syscall example in NASM. |
| `isa.md` | Complete | 4.4 KB | RISC vs. CISC, Arm vs. x86 history and market segmentation. |
| `layout.md` | Published | 11.1 KB | Machine code layout: CPU front-end (fetch+decode), code alignment, NOPs, instruction cache, unequal branches, `[[unlikely]]`, `cmov`. |
| `loops.md` | Complete | 5.8 KB | Loops, jumps, conditional jumps, FLAGS register, loop unrolling, alternative loop patterns (count-toward-zero). |

## Image Assets

4 images: `birthday.png` (45.9 KB — birthday paradox?), `clt.png` (88.2 KB — Central Limit Theorem?), `monte-carlo.gif` (180.0 KB), `pdf.png` (3.0 KB). These images seem misplaced — they relate to statistics/probability, not computer architecture. Likely belong to another chapter or were uploaded erroneously.

## Strengths

1. **Excellent progressive structure**: ISA → assembly → loops → functions → indirect branching → layout. Each builds on the previous.
2. **Arm/x86 dual perspective**: The assembly article shows both Arm and x86 for the same operation, giving a broader view.
3. **Practical focus**: The `layout.md` article connects code alignment, `[[likely]]`/`[[unlikely]]` attributes, and `cmov` directly to performance implications.
4. **Clear, compact explanations**: `functions.md` explains the stack, calling conventions, inlining, and tail calls in ~10KB — extremely efficient.
5. **Historical context**: The ISA article explains the RISC/CISC split and *why* the market segmented the way it did.
6. **Code-centric**: Nearly every concept is illustrated with assembly snippets.

## Areas for Improvement

1. **Misplaced image assets**: The 4 images in `img/` (birthday.png, clt.png, monte-carlo.gif, pdf.png) have nothing to do with computer architecture. They should be moved or removed.
2. **`interaction.md` is a stub**: Interrupts and system calls deserve more than a 747-byte draft. This is a fundamental concept that connects hardware to OS.
3. **No discussion of microarchitecture specifics**: While the chapter intentionally stays high-level, some mention of μops, ports, and execution units would bridge to the pipelining chapter more naturally.
4. **Missing topics**: (a) Endianness is discussed in `arithmetic/integer.md` but not here, (b) no explicit discussion of memory ordering/barriers, (c) no mention of speculative execution beyond a passing reference in `layout.md`.
5. **The ISA article is too brief**: At 4.4 KB, it only covers RISC vs. CISC. Could discuss instruction encoding, variable-length instructions, prefixes, and how ISA design affects performance.
6. **Unexplained forward references**: `layout.md` mentions "the pipeline of a CPU" from the pipelining chapter, but architecture comes *before* pipelining in the book order (weight 2 vs. weight 3). This could confuse sequential readers.

## Recommendations

1. **Move the misplaced images**: Find or create appropriate architecture diagrams (CPU block diagram, register file, stack frame diagram).
2. **Complete `interaction.md`**: Cover interrupts (hardware vs. software), exception handling, syscall overhead, the `syscall`/`sysenter` vs. `int 0x80` mechanisms, and the cost of context switches.
3. **Expand `isa.md`**: Add a section on instruction encoding, RISC-V as a modern clean-slate ISA, and the practical implications of ISA choices (e.g., why `cmov` exists, why fused instructions matter).
4. **Add microarchitecture preview**: A brief article or appendix on "What's inside a CPU core" (ALU, AGU, scheduler, register file, ROB) to prepare readers for pipelining.
5. **Reorder or note forward references**: Either move `layout.md` after pipelining, or add a brief forward-reference note explaining that pipeline details come in the next chapter.
6. **Consider adding**: An article on CPU privilege levels (ring 0/3), how `syscall` transitions between them, and why this matters for performance (system call overhead).
