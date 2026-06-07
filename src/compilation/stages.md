# Compilation Stages

Compilation is more than "gcc file.c." Understanding the stages helps you debug build problems, configure optimization, and use link-time optimization effectively.

## The Four Stages

### Stage 1: Preprocessing

The preprocessor (`cpp`) handles `#include`, `#define`, `#ifdef`, and related directives. It produces a single "translation unit" — the original source file with all headers expanded and all macros textually substituted.

```bash
gcc -E file.c -o file.i   # Preprocess only
```

What happens:
- `#include <stdio.h>` is replaced with ~20,000 lines of header content.
- `#define MAX 100` followed by `if (x > MAX)` becomes `if (x > 100)`.
- `#ifdef DEBUG` / `#endif` blocks are included or excluded.
- Comments are stripped.

The output is still C (or C++) source code, just with no preprocessing directives.

### Stage 2: Compilation (Translation)

The compiler proper (`cc1` for C, `cc1plus` for C++) translates the preprocessed source into assembly:

```bash
gcc -S file.i -o file.s    # Compile to assembly
gcc -S -O2 file.c -o file.s  # Compile with optimization, view assembly
```

This is where all the interesting work happens: parsing, semantic analysis, IR generation, optimization passes, instruction selection, register allocation. The output is assembly in the target ISA (x86-64, ARM64, etc.).

Viewing the assembly is a critical debugging tool. Always check that the compiler generated the instructions you expected:

```bash
gcc -S -O2 -march=native -fverbose-asm hot_loop.c -o hot_loop.s
```

### Stage 3: Assembly

The assembler (`as`) converts assembly text into machine code (an **object file**):

```bash
gcc -c file.s -o file.o    # Assemble
gcc -c file.c -o file.o    # Compile + assemble in one step
```

The object file contains:
- Machine code for each function.
- Data for global/static variables.
- A symbol table listing exported (`.globl`) and undefined symbols.
- Relocation entries: "fill in the address of `printf` once it's known."

Object files are in ELF format (Linux), Mach-O (macOS), or PE/COFF (Windows). You can inspect them:

```bash
objdump -d file.o          # Disassemble
objdump -t file.o          # Symbol table
nm file.o                  # Symbol names
readelf -a file.o          # Everything (ELF only)
```

### Stage 4: Linking

The linker (`ld`, invoked by `gcc`) combines multiple object files and libraries into a single executable:

```bash
gcc file1.o file2.o -lm -o program   # Link
```

The linker:
1. Resolves cross-references between object files (the call to `foo` in `file1.o` is patched to point to the definition of `foo` in `file2.o`).
2. Searches libraries (`-lm`, `-lpthread`) for unresolved symbols.
3. Assigns final addresses to all code and data sections.
4. Produces the executable in the platform's format.

## Static vs. Shared Libraries

**Static libraries** (`.a` on Linux, `.lib` on Windows): The library's object code is copied into the executable at link time. The executable is self-contained but larger. Static linking enables LTO across the library boundary.

**Shared libraries** (`.so` on Linux, `.dylib` on macOS, `.dll` on Windows): The executable stores only a reference to the library. The actual code is loaded at runtime by the dynamic linker (`ld.so`). Multiple processes share the same physical pages of library code. But:
- Function calls through shared library boundaries are indirect (through the PLT/GOT — Procedure Linkage Table / Global Offset Table).
- The compiler cannot inline across shared library boundaries.
- Library code is position-independent (PIC), which constrains some optimizations.

For performance-critical code that is distributed as a library, static linking (or LTO + static linking) gives better performance. For system libraries (`libc`, `libm`), shared is standard.

## Header-Only Libraries

A common C++ pattern: the entire library is in header files. Each user compiles the library code into their own translation unit. This enables:
- Full inlining across the "library" boundary.
- No ABI compatibility concerns.
- Template instantiation works naturally.

The cost: increased compile time (every TU recompiles the library) and potential code bloat. C++20 modules aim to address this.

## Link-Time Optimization (LTO)

Normally, the compiler optimizes one translation unit at a time. LTO defers some optimization passes to link time, when all translation units are visible:

```bash
gcc -flto -O2 file1.c file2.c -o program
```

With LTO, the compiler can:
- Inline across translation units.
- Propagate constants defined in one TU to callers in another.
- Eliminate dead functions and data across the whole program.
- Devirtualize virtual calls when the concrete type is known.

LTO typically improves performance by 5–15% for large programs. The cost is increased link time (the linker must run the optimizer on the whole program) and increased memory usage during linking.

**ThinLTO** (Clang/LLVM): A more scalable variant that partitions the program into modules, optimizes them in parallel, and cross-imports only the summary data needed for inter-module optimization. ThinLTO scales to programs with tens of millions of lines of code.

## Inspecting the Binary

Understanding what's in your binary helps diagnose size bloat, unexpected dependencies, and performance problems:

```bash
nm program | sort           # All symbols
objdump -t program          # Symbol table with sizes
objdump -d program | less   # Full disassembly
objdump -h program          # Section headers (text, data, rodata, bss sizes)
size program                # Section sizes summary
strip program               # Remove debug symbols (reduces size)
```

For performance work, the most useful command is `objdump -d` (or `gdb`'s `disassemble`) to see exactly what code was generated for a specific function.
