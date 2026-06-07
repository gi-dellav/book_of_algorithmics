# Link-Time Optimization

Link-Time Optimization (LTO) breaks down the barrier between translation units. It lets the compiler see the entire program — or at least large parts of it — and optimize across function and file boundaries. This article explains how LTO works, how to use it, and what it can and can't do.

## The Translation Unit Barrier

Traditionally, compilation works on one `.c` file at a time:

```bash
gcc -c file1.c -o file1.o  # Compiles file1.c, sees only file1.c
gcc -c file2.c -o file2.o  # Compiles file2.c, sees only file2.c
gcc file1.o file2.o -o prog  # Linker merges object files
```

When compiling `file1.c`, the compiler has no information about `file2.c`. If `file1.c` calls `foo()` which is defined in `file2.c`, the compiler must:
- Generate a generic call instruction.
- Assume nothing about what registers `foo()` clobbers.
- Assume `foo()` might modify any global variable.
- Assume `foo()` might throw an exception or call `longjmp`.

These conservative assumptions prevent many optimizations. The compiler can't inline `foo()`, can't propagate constants across the call, and can't eliminate the call even if `foo()` turns out to be empty.

## How LTO Works

With LTO, the compiler stores its intermediate representation (IR) in the object file instead of (or alongside) machine code:

```bash
gcc -flto -c file1.c -o file1.o  # Contains both IR and machine code
gcc -flto -c file2.c -o file2.o
gcc -flto file1.o file2.o -o prog  # Linker reads IR, optimizes, generates final code
```

At link time, the linker invokes the compiler's optimization passes on the combined IR of all files. The optimizer can now:
- Inline `foo()` from `file2.c` into callers in `file1.c`.
- Propagate constants defined in one file to uses in another.
- Eliminate dead functions that are never called.
- Devirtualize virtual calls where the concrete type is provable.

The final machine code is generated during the link step, using the combined knowledge of the entire program.

## What LTO Enables

### Cross-TU Inlining

```rust
// crate1/src/lib.rs
pub fn is_even(x: i32) -> bool { x % 2 == 0 }

// crate2/src/lib.rs
// Without LTO: is_even call cannot be inlined across crate boundary
// With LTO (lto = true in Cargo.toml): is_even can be inlined
use crate1::is_even;
fn sum_even(a: &[i32]) -> i32 {
    let mut s = 0;
    for i in 0..a.len() {
        if is_even(a[i]) { s += a[i]; }
    }
    s
}
```

Without LTO: `is_even` is a function call in the inner loop. With LTO: `is_even` is inlined, becoming `if (a[i] % 2 == 0)`, which the optimizer might further simplify with bitwise AND (`a[i] & 1`).

### Constant Propagation Across Files

```rust
// config.rs
pub const BLOCK_SIZE: usize = 256;

// main.rs
use config::BLOCK_SIZE;
fn process() {
    for i in 0..BLOCK_SIZE {  // Without LTO: load BLOCK_SIZE from memory
        // ...                   // With LTO: constant 256, unrolled
    }
}
```

### Dead Code Elimination

If a large library is linked but only a few functions are used, LTO can eliminate the unused functions from the binary entirely. Without LTO, the linker includes the entire object file when any symbol from it is referenced (with `-ffunction-sections -fdata-sections -Wl,--gc-sections` as a partial mitigation).

## ThinLTO

Full LTO serializes the entire program's IR into a single module, which is then optimized as a monolith. This is expensive: a program with 10 million lines of code produces gigabytes of IR, and the optimization passes take minutes and consume tens of gigabytes of RAM.

ThinLTO (Clang/LLVM) addresses this by:
1. Each translation unit is compiled to IR + a summary (which functions it defines, which it calls, what constants it provides).
2. At link time, the summaries are merged. Cross-module inlining candidates are identified.
3. Each module is optimized independently, importing only the IR for the functions it needs to inline.
4. A final lightweight link pass combines the optimized modules.

ThinLTO scales linearly: a 100-million-line program can be optimized with ThinLTO in roughly the same time per module as non-LTO compilation, with memory usage proportional to the largest module, not the whole program.

GCC's equivalent is `-flto=auto` (parallel LTO) or using the `lto` wrapper with `make -j`.

## Enabling LTO

```bash
# GCC
gcc -flto -O2 -c file1.c file2.c file3.c
gcc -flto -O2 file1.o file2.o file3.o -o prog

# Clang (Full LTO)
clang -flto -O2 -c file1.c file2.c file3.c
clang -flto -O2 file1.o file2.o file3.o -o prog

# Clang (ThinLTO)
clang -flto=thin -O2 -c file1.c file2.c file3.c
clang -flto=thin -O2 file1.o file2.o file3.o -o prog
```

Or, for single-command compilation:
```bash
gcc -flto -O2 file1.c file2.c file3.c -o prog
```

## LTO and Static Libraries

When linking against static libraries, LTO can inline library functions into your code:

```bash
# Build the library with LTO
gcc -flto -c libfoo.c
ar rcs libfoo.a libfoo.o

# Link with LTO
gcc -flto -O2 main.c -L. -lfoo -o prog
```

The `.a` file contains IR, and the linker extracts and optimizes it alongside your code.

This does NOT work with shared libraries (`.so`), because shared library code is not available at link time. LTO requires seeing the IR of all involved code at link time.

## LTO and Build Systems

CMake example:
```cmake
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
# Or for ThinLTO with Clang:
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)
set(CMAKE_C_COMPILER clang)
add_link_options(-flto=thin)
```

Makefile example:
```makefile
CFLAGS = -O2 -flto
LDFLAGS = -O2 -flto
# Ensure the linker uses the same compiler for LTO
CC = gcc
```

The critical rule: **use the same compiler and flags for both compilation and linking.** The linker invokes the compiler's LTO plugin; mismatched versions can produce strange errors.

## Performance Impact

Typical improvements from enabling LTO:

- **Integer/pointer-heavy code**: 3–8% faster. Most benefit from cross-TU inlining of small utility functions.
- **C++ template-heavy code**: 5–15% faster. Template instantiations in different TUs can be merged and devirtualized.
- **Code with globals**: 5–10% faster. Constant globals get propagated; dead globals are eliminated.
- **Code size**: Usually smaller (dead code elimination + more aggressive inlining of small functions). Occasionally larger (over-aggressive inlining).

## Debugging LTO

LTO can make debugging harder: the generated code doesn't correspond cleanly to source files, and optimized builds already obscure the source-to-assembly mapping. For debugging with LTO:

```bash
gcc -flto -O2 -g ...  # Include debug info
```

GDB can usually handle LTO binaries, but stack traces may show "inlined" frames and variables may be optimized away.

If you suspect an LTO bug:
1. Verify the bug disappears without LTO (`-fno-lto`).
2. Try reducing the optimization level (`-O1 -flto` vs. `-O2 -flto`).
3. Isolate the problematic translation unit by compiling only it without LTO.
4. Report the bug with a reduced test case.
