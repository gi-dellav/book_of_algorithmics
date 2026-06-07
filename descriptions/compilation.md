# Chapter: Compilation (`compilation/`)

## Overview

Weight 4 in the book. This chapter bridges assembly knowledge and practical compiler usage. It covers compilation stages, optimization flags, situational optimizations (loop unrolling, inlining, branch hints, PGO), contract programming (undefined behavior, `restrict`, assumptions), precomputation via `constexpr`, and the limitations of compilers. The `_index.md` frames the goal as "getting the compiler to do exactly what we want."

## Files and Content

| File | Status | Size | Description |
|------|--------|------|-------------|
| `_index.md` | Complete | 888 B | Introduction: the value of understanding what the compiler does |
| `abstractions.md` | Draft | 3.2 KB | Non-zero-cost abstractions: virtual functions, `std::min` overhead, pointer-chasing containers, and a cache-oblivious matrix wrapper example |
| `arithmetic.md` | Draft/Stub | 68 B | Empty placeholder for "Arithmetic Optimizations" |
| `contracts.md` | Complete | 11.5 KB | Undefined behavior (why it exists, signed overflow example), assumptions (`__builtin_assume`, `__builtin_unreachable`), memory aliasing (`restrict`, `const`), and C++ contracts proposal |
| `flags.md` | Published | 3.4 KB | Optimization levels (`-O0` through `-Ofast`), target specification (`-march`, `-mtune`), pragmas, and multiversioned functions |
| `limitations.md` | Draft | 3.4 KB | What compilers can and can't do; a checklist for diagnosing optimization failures |
| `precalc.md` | Complete | 3.1 KB | Compile-time computation: `constexpr` functions, limitations, constexpr arrays for lookup tables, fibonacci example |
| `situational.md` | Complete | 5.0 KB | Situational optimizations: loop unrolling pragmas, `always_inline`, `[[likely]]`/`[[unlikely]]`, and profile-guided optimization (PGO) |
| `stages.md` | Complete | 4.7 KB | The 4 stages (preprocess, compile, assemble, link), LTO, static vs. shared libraries, header-only libraries, inspecting assembly output |

## Strengths

1. **Pragmatic advice throughout**: The chapter consistently gives actionable guidance — use `-O3` and `-march=native`, check assembly, use `restrict`, run PGO.
2. **`contracts.md` is excellent**: The explanation of *why* signed integer overflow is UB (to enable optimizations like `(x+1) > x` always being true) is a rare and valuable insight. The step-by-step walkthrough of division-by-2 for signed vs. unsigned is concrete and memorable.
3. **Good coverage of modern C++**: `constexpr`, `[[likely]]`, contract attributes proposal, and `__builtin_unreachable` are all covered.
4. **PGO explanation is practical**: Clear shell commands showing the two-pass compilation process with `-fprofile-generate` and `-fprofile-use`.
5. **The "limitations" checklist**: The four questions to ask when an optimization doesn't happen is a useful debugging framework.

## Areas for Improvement

1. **`arithmetic.md` is empty**: The "Arithmetic Optimizations" article at weight 10 is a stub. Given that the `arithmetic/` chapter already exists, this might be redundant — perhaps it should be removed or repurposed.
2. **`abstractions.md` is rough**: The draft mixes critique of `std::min`, virtual functions, and bounds checking with a cache-oblivious transpose example that feels out of place. Needs restructuring or splitting.
3. **Missing topics**: (a) Link-time optimization (LTO) is mentioned in `stages.md` but not given a dedicated article — LTO deserves more detail given its importance for cross-TU inlining, (b) no discussion of sanitizers (ASan, UBSan, TSan) which are crucial for catching UB before relying on it for optimization, (c) no coverage of compiler intrinsics for accessing hardware features (e.g., `__builtin_popcount`).
4. **`limitations.md` feels incomplete**: The checklist is good but the article doesn't include concrete examples of optimization failures and how to resolve them.
5. **No discussion of build system optimization**: ccache, distcc, unity builds, precompiled headers — relevant for the "compilation time" aspect mentioned in `flags.md`.
6. **GCC-centric**: Clang equivalents are mentioned sporadically but not systematically. MSVC is not mentioned at all.

## Recommendations

1. **Remove or repurpose `arithmetic.md`**: Since `arithmetic/` is a separate chapter, this stub is misleading. Either delete it or replace with a cross-reference article pointing to relevant techniques.
2. **Expand `abstractions.md`**: Split into (a) abstraction overhead case studies with benchmarks (virtual functions, `std::function`, exceptions, RTTI), and (b) designing abstraction-friendly interfaces that don't sacrifice performance.
3. **Add a dedicated LTO article**: Cover ThinLTO vs. full LTO, how to enable it, what optimizations it enables that regular compilation can't, and the build time tradeoffs.
4. **Add a sanitizers article**: UB sanitizer, address sanitizer, and how to use them in conjunction with optimization (catch bugs in debug, then exploit the guarantees in release).
5. **Complete `limitations.md`**: Add 3-4 concrete case studies where the compiler fails to optimize and exactly why (aliasing, side effects, insufficient information, missing language features).
6. **Add a "Compiler Explorer" section**: A brief tutorial on using godbolt.org effectively — how to isolate functions, compare compilers, and interpret the color-coded output.
