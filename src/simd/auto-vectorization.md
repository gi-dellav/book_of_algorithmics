# Auto-Vectorization

The easiest SIMD is the one you don't write. Modern compilers can automatically transform scalar loops into SIMD instructions when the loop meets certain criteria. This article covers how to write loops the compiler can vectorize, and how to debug when it doesn't.

## What Auto-Vectorization Requires

For the compiler to vectorize a loop, it must be able to prove:

1. **The iteration count is known** (or at least bounded). `for (int i = 0; i < n; i++)` — good. `while (*p)` — generally not vectorizable.

2. **No dependencies between iterations.** Each iteration must read from and write to independent locations.

3. **No aliasing between pointers.** If `a`, `b`, and `c` might overlap, the compiler must generate scalar code.

4. **Contiguous memory access.** `a[i]` is vectorizable; `a[i * stride]` with stride > 1 may or may not be (gather instructions for variable stride, or strided access with `vgather`).

5. **Loop body is simple enough.** No function calls, no complex control flow, no I/O.

## Enabling Auto-Vectorization

```bash
gcc -O2 -march=native -ftree-vectorize  # -O2 enables auto-vec; -O3 is more aggressive
```

To see which loops were vectorized (and which weren't, and why):

```bash
gcc -O2 -march=native -fopt-info-vec -fopt-info-vec-missed file.c
```

Output:
```
file.c:12:3: note: loop vectorized
file.c:25:5: note: not vectorized: data dependency prevents vectorization
file.c:36:8: note: not vectorized: number of iterations cannot be computed
```

## Common Vectorization Blockers

### Unknown Iteration Count

```rust
// Not vectorized: n is unknown, could be 0 or very small
fn add(a: &[f32], b: &[f32], c: &mut [f32]) {
    for i in 0..c.len() {
        c[i] = a[i] + b[i];
    }
}
```

Fix: Use `unsigned` or `size_t` so the compiler knows n ≥ 0. Still not perfect — the compiler must generate a remainder loop for `n % 8 != 0`.

```rust
fn add(a: &[f32], b: &[f32], c: &mut [f32]) {
    for i in 0..a.len() {
        c[i] = a[i] + b[i];
    }
}
```

### Aliasing

```rust
fn scale_and_add(a: &mut [f32], b: &[f32], s: f32) {
    for i in 0..a.len() {
        a[i] = b[i] * s + a[i];  // 'a' and 'b' might alias
    }
}
```

Fix: add `restrict`.

```rust
fn scale_and_add(a: &mut [f32], b: &[f32], s: f32) {
    // In Rust, &mut and & references are guaranteed not to alias
    for i in 0..a.len() {
        a[i] = b[i] * s + a[i];
    }
}
```

### Dependencies Between Iterations

```rust
// Not vectorized: x[i] depends on x[i-1]
for i in 1..n {
    x[i] = x[i-1] + y[i];
}
```

This is a prefix sum (scan) — a true data dependency. Special algorithms exist for SIMD prefix sums (see `algorithms/prefix.md`), but the compiler won't vectorize it automatically.

### Non-Contiguous Access

```rust
// Strided access: the compiler may vectorize with gather (AVX2+) but it's slower
for i in 0..n {
    sum += a[i * stride];
}
```

If `stride` is a compile-time constant and small (2, 4), the compiler may use `shuffle` + `add` sequences. For large or variable stride, gather instructions (`vgatherdps`) are slow — often slower than scalar.

## SPMD Model: Julia, OpenMP, ISPC

Auto-vectorization is fragile. An alternative: explicitly write in a "Single Program, Multiple Data" style where you describe the operation for one element and the compiler maps it to SIMD lanes:

**Julia**:
```julia
@simd for i in 1:n
    c[i] = a[i] + b[i]
end
```

**OpenMP**:
```rust
// Use iterators for potential auto-vectorization by LLVM
for i in 0..n {
    c[i] = a[i] + b[i];
}
```

**ISPC** (Intel SPMD Program Compiler): A dedicated language for writing SIMD kernels that compiles to AVX2/AVX-512/NEON. Used in game engines and renderers.

## Debugging Auto-Vectorization Failures

1. Compile with `-fopt-info-vec-missed` and read every warning.
2. Check the assembly (`-S`) for the loop — look for `vaddps`, `vmulps`, etc. (SIMD) vs. `addss`, `mulss` (scalar).
3. Add `restrict`.
4. Use `#pragma GCC ivdep` to tell the compiler to ignore assumed vector dependencies (dangerous — wrong if dependencies exist).
5. Use `__builtin_assume(n % 8 == 0)` to tell the compiler the trip count is a multiple of the vector width.
6. Consider intrinsics — if the compiler can't figure it out, write the SIMD explicitly.
