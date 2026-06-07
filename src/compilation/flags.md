# Optimization Flags

The compiler has hundreds of knobs. This article catalogs the ones that matter for performance, focusing on GCC and Clang (MSVC equivalents noted where relevant).

## Optimization Levels

```bash
-O0    # No optimization (debug default)
-Og    # Optimizations that don't interfere with debugging
-O1    # Conservative optimization
-O2    # Standard optimization (release default)
-O3    # Aggressive optimization
-Os    # Optimize for size (-O2 minus size-increasing opts)
-Ofast # -O3 + -ffast-math + more (may violate standards)
-Oz    # Aggressively optimize for size (Clang only)
```

**Recommendation**: Use `-O2` as your baseline. Try `-O3` and measure — it's usually faster but occasionally slower due to loop unrolling and aggressive inlining. Use `-Ofast` only if you understand the floating-point implications.

## Target Architecture

```bash
-march=native         # Use all ISA extensions available on the compiling machine
-march=x86-64-v3      # x86-64 microarchitecture level 3 (AVX2, BMI, FMA, etc.)
-march=znver3         # Target Zen 3 (Zen 2: znver2, Zen 4: znver4)
-mtune=native         # Optimize scheduling for this CPU, but don't use new instructions
```

x86-64 microarchitecture levels (for portable binaries):
- **x86-64** (baseline): SSE2, CMOV, FXSR. ~2003 and later.
- **x86-64-v2**: SSE4.2, SSSE3, POPCNT, CMPXCHG16B. ~2008 (Intel Nehalem).
- **x86-64-v3**: AVX2, BMI1/2, FMA, LZCNT, MOVBE. ~2013 (Intel Haswell).
- **x86-64-v4**: AVX-512F, AVX-512BW/DQ/VL, etc. ~2017 (Intel Skylake-X).

Building with `-march=x86-64-v2` gives most integer performance benefits while maintaining broad compatibility. AVX2 (`-v3`) is needed for SIMD optimization.

## Floating-Point Flags

```bash
-ffast-math           # Enable all -funsafe-math-optimizations flags
```

`-ffast-math` is a macro that enables:
- `-ffinite-math-only`: Assume no NaNs or infinities.
- `-fno-signed-zeros`: Ignore sign of zero.
- `-fno-trapping-math`: Assume no floating-point exceptions.
- `-funsafe-math-optimizations`: Allow reassociation (`(a+b)+c` → `a+(b+c)`), reciprocal transformations (`a/b` → `a*(1/b)`).
- `-fno-math-errno`: Don't set `errno` on math errors.
- `-freciprocal-math`: Use reciprocals for division.

The key benefit for performance is reassociation (enables auto-vectorization of reductions) and FMA contraction (`a*b+c` → `fma(a,b,c)` with one rounding instead of two).

**Tradeoff**: Results may differ from IEEE 754 in the last bit or two. NaN propagation changes. For most algorithms, this is acceptable; for scientific computing with careful error analysis, it may not be.

If you need more control than `-ffast-math`, enable individual flags:

```bash
-fno-signed-zeros      # Safe for most code
-ffinite-math-only      # Safe if your data has no NaN/Inf
-fno-trapping-math      # Safe unless you rely on FP exceptions
-mfma                   # Enable FMA instructions (safe, improves accuracy)
```

## Inlining Flags

```bash
-finline-functions       # Enable inlining of non-inline functions (-O2+)
-finline-limit=N         # Set inlining size threshold
-finline-small-functions # Inline small functions even at -O1
```

Inlining is the most impactful single optimization. The compiler uses heuristics to decide what to inline. You can override with attributes on specific functions.

## Loop Optimization Flags

```bash
-funroll-loops           # Unroll loops where beneficial (-O3)
-funroll-all-loops       # Unroll all loops (often harmful — increases I-cache pressure)
-fprefetch-loop-arrays   # Insert prefetch instructions in loops (-O3)
-ftree-vectorize         # Auto-vectorize loops (-O2+)
-fvect-cost-model=unlimited # More aggressive vectorization
-fno-semantic-interposition # Assume functions can't be interposed (LTO substitute)
```

## PGO (Profile-Guided Optimization)

PGO collects runtime profile data and feeds it back to the compiler:

```bash
# Step 1: Instrumented build
gcc -O2 -fprofile-generate -o program program.c

# Step 2: Run with representative workload
./program training_input_1
./program training_input_2

# Step 3: Rebuild using profile data
gcc -O2 -fprofile-use -o program program.c
```

PGO improves:
- Branch prediction layout (hot path sequential, cold path out-of-line).
- Inlining decisions (inline frequently-called small functions, avoid inlining cold functions).
- Loop unrolling decisions.
- Function ordering (hot functions close together → fewer I-cache misses).

Typical improvement: 5–15% on integer code, 10–20% on branch-heavy code. The cost is the two-pass build and the need for representative training data.

## Sanitizers

Sanitizers detect bugs at runtime. They're invaluable for catching undefined behavior that could corrupt your optimization assumptions:

```bash
-fsanitize=undefined     # UB sanitizer: signed overflow, null deref, misaligned access
-fsanitize=address       # Address sanitizer: out-of-bounds, use-after-free, double-free
-fsanitize=thread        # Thread sanitizer: data races
-fsanitize=memory        # Memory sanitizer: uninitialized reads (Clang only)
```

Run sanitizers on your test suite in a debug build. Fix all reported issues before relying on optimization assumptions. UB that "seems to work" at `-O0` may produce wrong results at `-O2` when the compiler exploits it for optimization.

## Recommended Release Flags

```bash
# Baseline (portable to any x86-64):
-O2 -march=x86-64-v2

# Performance (for known hardware):
-O3 -march=native -flto

# Numerical (if FP semantics allow):
-O3 -march=native -flto -ffast-math

# With PGO:
-O3 -march=native -flto -fprofile-use
```

Always measure the effect of each flag. The interaction between optimizations is complex, and `-O3` is not universally faster than `-O2`.
