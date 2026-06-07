# What the Compiler Does

The gap between the code you write and the code that executes is filled by the compiler. For most programmers, the compiler is a black box: source code in, binary out. For performance work, that's not good enough. You need to know what the compiler can do for you, what it can't, and how to persuade it to do more.

## The Compiler's Job

A modern optimizing compiler (GCC, Clang, MSVC) does far more than translate C to assembly. It:

1. **Resolves names and types** (semantic analysis).
2. **Lowers abstractions** — loops become jumps, classes become structs with function pointers, lambdas become anonymous classes.
3. **Applies generic optimizations** — constant folding, dead code elimination, common subexpression elimination, strength reduction.
4. **Applies target-specific optimizations** — instruction selection, register allocation, scheduling for the pipeline, auto-vectorization.
5. **Emits object code** in the target format (ELF, Mach-O, PE/COFF).

At `-O0`, the compiler does step 1 and 2, then emits straightforward (often naive) code. At `-O2` and above, it applies hundreds of optimization passes, each rewriting the intermediate representation (IR) into a more efficient form.

## Optimization Levels

| Flag | Meaning | Notes |
|------|---------|-------|
| `-O0` | No optimization | Debuggable; every variable has a stack location; statements map directly to assembly. |
| `-O1` | Basic optimization | Inlining, basic dead code elimination, constant propagation. Good for debugging with decent performance. |
| `-O2` | Standard optimization | Most optimizations enabled. The default for release builds. ~2–5× faster than `-O0`. |
| `-O3` | Aggressive optimization | Adds `-funroll-loops`, `-fipa-cp`, more aggressive inlining. Usually faster, sometimes larger. |
| `-Os` | Optimize for size | Like `-O2` but disables optimizations that increase code size. |
| `-Ofast` | Disregard standards compliance | `-O3` + `-ffast-math` + a few others. Faster but may break correctness. |

The biggest jump is from `-O0` to `-O2`. The jump from `-O2` to `-O3` is smaller and sometimes negative (loop unrolling can increase I-cache pressure). Measure both.

## Target Selection

By default, GCC and Clang generate code that runs on any x86-64 processor (baseline: SSE2, no AVX, no BMI). This means they cannot use instructions introduced after ~2003.

```bash
gcc -march=native    # Use all instructions available on this CPU (AVX2, BMI, POPCNT, etc.)
gcc -march=znver2    # Target Zen 2 specifically
gcc -mtune=znver2    # Optimize for Zen 2's pipeline, but don't use ISA extensions
```

`-march=native` enables:
- AVX/AVX2 for SIMD
- BMI1/BMI2 for bit manipulation (`andn`, `blsi`, `pext`, `pdep`)
- POPCNT, LZCNT, TZCNT for bit counting
- FMA for fused multiply-add
- AESNI, SHA for cryptography
- And dozens of other extensions

These can double or triple performance for data-parallel code. Always use `-march=native` for benchmarks, and for binaries shipped to known hardware. Use `-mtune=generic -march=x86-64-v2` (or `-v3`, `-v4`) for portable binaries.

## The Intermediate Representation

Compilers operate on an IR between the source and the target:

- **GCC**: GIMPLE → RTL (Register Transfer Language)
- **Clang/LLVM**: LLVM IR → SelectionDAG → MachineIR

The IR is a simplified, SSA-form (Static Single Assignment) representation. Each variable is assigned exactly once. This makes dataflow analysis (tracking where values come from and go to) tractable.

Example: `x = a + b; y = x * 2;` becomes:
```
%1 = add i32 %a, %b
%2 = mul i32 %1, 2
```

Each `%N` is an SSA "virtual register." The compiler can see that `%1` flows into `%2` and that `%1` has no other uses (can be eliminated after `%2` is computed).

## What the Compiler Can Do

The compiler can perform optimizations that would be tedious or impossible by hand:

- **Constant propagation**: `x = 2; y = x + 3;` → `y = 5;`.
- **Dead code elimination**: If `x` is never used after being computed, the computation is deleted.
- **Common subexpression elimination**: `a = b*c + d; e = b*c + f;` → `t = b*c; a = t + d; e = t + f;`.
- **Strength reduction**: `x * 8` → `x << 3`; `x / 8` → `x >> 3` (for unsigned).
- **Inlining**: Replacing a function call with the callee's body.
- **Loop-invariant code motion**: Moving `sqrt(n)` out of a loop if `n` doesn't change.
- **Induction variable optimization**: `for (i=0; i<n; i++) a[i] = i*4;` → `for (p=a; p<a+n; p++) *p = (p-a)*4;` (transforms array indexing to pointer arithmetic).
- **Auto-vectorization**: Translating scalar loops to SIMD instructions.

## What the Compiler Cannot Do

The compiler cannot:
- **Violate the language standard**: If the standard says signed overflow is undefined, the compiler can *exploit* that, but it can't change the observable behavior of a correctly-written program.
- **Change floating-point semantics** (without `-ffast-math`): `(a+b)+c` must round after each addition; the compiler can't reassociate to `a+(b+c)` even if that's mathematically equivalent.
- **Optimize across translation units** (without LTO): Functions in different `.c` files are opaque at compile time. The compiler can't inline them or propagate constants across them.
- **Resolve pointer aliasing**: If two pointers *might* point to the same location (`restrict` not used), the compiler must generate code that works for both aliasing and non-aliasing cases.
- **Know runtime values**: The compiler can't optimize for a specific input size. For `n = 100`, a sequential search might beat binary search, but the compiler must handle arbitrary `n`.

Understanding these limitations — especially aliasing, floating-point semantics, and cross-TU optimization — is the key to writing "compiler-friendly" code.
