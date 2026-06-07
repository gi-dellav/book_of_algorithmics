# ISA: RISC vs. CISC

The **Instruction Set Architecture** (ISA) is the contract between hardware and software. It specifies: the instructions the CPU will execute, the registers available, the addressing modes, the data types, and the memory model. Software above the ISA (your C program, the OS kernel) is portable across implementations. Hardware below the ISA (the microarchitecture) can change radically while running the same software.

Two ISAs dominate modern computing: **x86-64** (Intel, AMD) in desktops, laptops, and servers; and **ARM** in phones, tablets, and increasingly laptops (Apple M1/M2) and servers (AWS Graviton). A third, **RISC-V**, is an open standard gaining traction in embedded systems and research.

## RISC vs. CISC

The distinction goes back to the 1980s, and it was as much a business battle as a technical one.

### CISC: Complex Instruction Set Computer

Exemplified by x86. The philosophy: give the programmer powerful, expressive instructions that do a lot of work in one operation. The canonical example is `rep movsb` — "repeat move string byte" — which copies a block of memory from source to destination with a single instruction prefix. The hardware handles the looping internally.

Characteristics:
- Variable-length instructions (x86: 1 to 15 bytes).
- Many addressing modes (register, immediate, direct, indirect, indexed, based, scaled-index).
- Instructions that combine memory access with arithmetic (e.g., `add eax, [ebx+esi*4+offset]` — load from memory, scale by 4, add offset, add to register).
- Complex instructions that perform multi-step operations (string operations, `enter`/`leave` for stack frames, trigonometric instructions in x87 FPU).

The cost: variable-length instructions are hard to decode in parallel. Complex instructions may take many cycles and complicate pipelining. The x86 front-end must parse a byte stream, identify instruction boundaries, and translate into internal micro-operations.

### RISC: Reduced Instruction Set Computer

Exemplified by ARM and RISC-V. The philosophy: keep instructions simple, fixed-size, and orthogonal. Let the compiler do the work of combining simple instructions into complex sequences.

Characteristics:
- Fixed-length instructions (ARM: 4 bytes, with optional 2-byte Thumb mode).
- Load-store architecture: arithmetic operates only on registers, not memory. Separate `load` and `store` instructions move data between registers and memory.
- Many registers (ARM: 31 general-purpose registers; x86-64: 16).
- Few addressing modes (typically base+offset and base+index).
- No complex multi-cycle instructions.

The benefit: simpler decode, easier pipelining, more predictable timing. The cost: RISC code is typically larger (more instructions to do the same work), though fixed-length instructions mean the decode logic is much simpler.

### The Convergence

The RISC/CISC distinction has blurred to near-irrelevance in modern high-performance designs. Today:

- x86 processors decode CISC instructions into **µops** (micro-operations) — simple RISC-like operations that are scheduled, executed, and retired by the out-of-order engine. The "CISC" part is just the front-end.
- ARM processors have added complex instructions (SIMD, cryptography extensions, load/store pair) where they provide a clear performance benefit.
- Both families use deep pipelines, out-of-order execution, branch prediction, and cache hierarchies. The differences are in the details, not the philosophy.

## The x86-64 ISA

x86-64 (also called AMD64 or Intel 64) is a 64-bit extension of the 32-bit x86 ISA, which itself extends the 16-bit 8086 from 1978. The lineage is visible in the ISA's quirks.

**Registers**: 16 general-purpose 64-bit registers: `rax`, `rbx`, `rcx`, `rdx`, `rsi`, `rdi`, `rbp`, `rsp`, `r8`–`r15`. The low 32 bits of each are `eax`, `ebx`, etc. The low 16 bits are `ax`, `bx`, etc. The low 8 bits of the first four are `al`, `bl`, `cl`, `dl`; the high 8 bits of the 16-bit registers are `ah`, `bh`, `ch`, `dh` (a legacy quirk).

**Addressing**: `[base + index*scale + displacement]` where base and index are registers, scale is {1,2,4,8}, and displacement is an immediate constant. You can omit any component. This rich addressing mode often lets a `load` and an `add` fuse into a single instruction: `mov rax, [rbx + rcx*8 + 16]`.

**Instruction encoding**: Variable-length, from 1 byte (`ret`, `push rax`) to 15 bytes. Prefix bytes modify behavior (REX for 64-bit, VEX/EVEX for AVX/AVX-512, LOCK for atomic operations). The opcode follows, then ModR/M (addressing mode), SIB (scale-index-base), and displacement/immediate bytes.

**SIMD**: SSE (128-bit, 16 registers `xmm0`–`xmm15`), AVX/AVX2 (256-bit, 16 registers `ymm0`–`ymm15`), AVX-512 (512-bit, 32 registers `zmm0`–`zmm31`). SIMD is covered in Chapter 10.

## The ARM64 ISA (AArch64)

A clean-slate 64-bit design (2011), dropping much of the legacy of 32-bit ARM. Apple's M1/M2 chips and AWS Graviton run ARM64.

**Registers**: 31 general-purpose 64-bit registers `x0`–`x30`. `xzr` is the zero register (always reads as 0, writes are ignored). `sp` is the stack pointer (not a general-purpose register). `x30` is the link register (return address). The low 32 bits are `w0`–`w30`.

**Addressing**: Base + immediate offset, or base + register offset (optionally scaled). Load-store architecture: `ldr x0, [x1, #8]` (load from x1+8 into x0), `str x0, [x1, x2, lsl #3]` (store x0 to x1 + x2*8).

**Instruction encoding**: Fixed 32-bit instructions. Clean, regular encoding simplifies decode. ARM64 can decode more instructions per cycle with less power than x86.

**SIMD**: NEON (128-bit, 32 registers `v0`–`v31`). SVE (Scalable Vector Extension) for wider vectors, vector-length-agnostic programming.

## Why the ISA Matters for Performance

1. **SIMD width**: Wider vectors mean more work per instruction. AVX-512 processes 16 floats per operation; NEON processes 4. For data-parallel code, the ISA directly determines throughput.
2. **Register count**: More registers means fewer spills to the stack. ARM64's 31 registers vs. x86-64's 16 can make a noticeable difference in register-pressure-heavy code.
3. **Addressing modes**: x86's complex addressing modes allow fusing a load with arithmetic, saving one instruction per memory access. ARM needs a separate `ldr` instruction.
4. **Instruction encoding density**: x86's variable-length encoding can be more compact (smaller code → less I-cache pressure). ARM's fixed-length encoding is easier to decode (wider decode → more instructions per cycle).
5. **Specialized instructions**: Both ISAs have added instructions for cryptography (AES, SHA), string operations, and bit manipulation. Using them can be 10× faster than software implementation.

The rest of this chapter focuses on x86-64 because it's the most widely used ISA in performance-critical computing, but we'll note ARM64 equivalents where the differences are instructive.
