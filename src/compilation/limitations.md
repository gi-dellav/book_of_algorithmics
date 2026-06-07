# Compiler Limitations

Compilers are not magic. They operate under strict constraints: they must preserve the observable behavior of your program, they have limited information about runtime values, and they have finite time budgets for optimization. This article catalogs what compilers can't do and what you should check when an optimization fails.

## The Optimization Failure Checklist

When you expect the compiler to optimize something and it doesn't, ask:

### 1. Is there aliasing?

```rust
fn scale_add(a: &mut [f32], b: &[f32], c: &[f32], s: f32) {
    for i in 0..a.len() {
        a[i] = b[i] * s + c[i];  // In Rust, &mut guarantees no aliasing with & refs
    }
}
```

If `a`, `b`, and `c` might overlap, the compiler cannot:
- Vectorize the loop (write to `a[i]` might affect `b[i+1]`).
- Reorder the iterations.
- Keep values in registers across iterations.

Fix: add `restrict`.

### 2. Is the compiler constrained by floating-point rules?

```rust
let mut sum = 0.0f32;
for i in 0..n {
    sum += a[i];  // Reassociation would require relaxed float semantics
}
```

Without `-ffast-math`, the compiler cannot reassociate `(a+b)+c` to `a+(b+c)`, which prevents many vectorization and ILP optimizations.

Fix: add `-ffast-math` or at least `-fno-signed-zeros -ffinite-math-only`.

### 3. Is there a function call with side effects?

```rust
for i in 0..n {
    a[i] = expensive_computation(i);  // Can the compiler hoist this?
}
```

If `expensive_computation` has side effects (modifies global state, does I/O), the compiler cannot eliminate or reorder calls. If it's pure (no side effects, always returns the same value for the same inputs), mark it with `__attribute__((const))` (GCC/Clang) or `[[gnu::const]]` (C++).

```rust
// Rust has no direct equivalent of __attribute__((const));
// use LTO or ensure the function body is visible to the compiler
fn expensive_computation(x: i32) -> i32;
```

### 4. Is the loop bound unknown?

```rust
fn process(a: &mut [f32]) {
    for i in 0..a.len() {
        a[i] = /* ... */;
    }
}
```

If `n` is not known at compile time, the compiler must handle the general case, including `n < 8` (can't unroll by 8) and `n % 8 != 0` (needs remainder loop). If you know `n` is a multiple of 8 and ≥ 8, communicate it:

```rust
if n % 8 != 0 { unsafe { std::hint::unreachable_unchecked() } }
if n < 8 { unsafe { std::hint::unreachable_unchecked() } }
```

Or use `__builtin_assume(n % 8 == 0)` (Clang).

### 5. Is there a memory dependency the compiler can't prove is absent?

```rust
fn copy(dst: &mut [u8], src: &[u8]) {
    for i in 0..dst.len() {
        dst[i] = src[i];  // &mut and & references are guaranteed non-aliasing
    }
}
```

If `dst` and `src` might point into the same buffer with an offset less than the loop's vector width, the compiler must generate scalar code. `restrict` fixes this if aliasing never occurs. If it does occur, `memmove` handles overlap correctly.

### 6. Did the compiler simply run out of optimization budget?

Compilers have internal cutoffs: maximum unroll factor, maximum inlining depth, maximum number of expressions to track for common subexpression elimination. Extremely large functions or deeply nested loops may exceed these limits. Split large functions into smaller ones.

## Concrete Failure Cases

**Case 1: The compiler won't vectorize a simple loop**

```rust
fn copy(a: &mut [i32], b: &[i32]) {
    for i in 0..a.len() {
        a[i] = b[i];
    }
}
```

Check: `gcc -O2 -fopt-info-vec-missed` shows:
```
note: not vectorized: number of iterations cannot be computed.
note: not vectorized: control flow in loop.
```

Fix: change `int n` to `unsigned n` (or `size_t n`) so the compiler knows `n ≥ 0` and the iteration count is well-defined. Or add `-fno-tree-loop-distribute-patterns`.

**Case 2: Function call prevents optimization**

```rust
fn foo(x: f64) -> f64 { bar(x) }  // Cannot inline, bar is in another crate
```

Fix: use LTO (`-flto`), or make `bar` available in a header as `static inline`.

**Case 3: Dead store elimination fails**

```rust
fn memset_zero(buf: &mut [u8]) {
    for byte in buf.iter_mut() {
        *byte = 0;
    }
}
```

The compiler may eliminate this store entirely if it can prove `buf` is never subsequently read. But if `buf` is passed through a function boundary, the compiler must conservatively keep the store. This is usually correct behavior — if `buf` is a buffer being passed to a crypto function, eliminating the zeroing would be a security vulnerability. Use `__asm__ volatile("" : : "r"(buf) : "memory")` or `secure_memzero` from your crypto library if you need guaranteed zeroing.

## Using Compiler Explorer

[godbolt.org](https://godbolt.org) is invaluable for diagnosing optimization failures. The workflow:

1. Paste your function into the left pane.
2. Select your compiler (`x86-64 gcc 13.2` or `x86-64 clang 17.0`).
3. Set flags (`-O2 -march=native`).
4. Read the assembly in the right pane.
5. If the assembly isn't what you expect, tweak the source or flags until it is.

Pro tips:
- Color-coded lines map source to assembly. Hover over a source line to see the corresponding instructions.
- Add `-fverbose-asm` to see which C variables map to which registers.
- Add `-fopt-info-vec` and `-fopt-info-vec-missed` to see vectorization diagnostics in the output pane.
- Compare GCC and Clang — each has different strengths.
- Use the "diff" feature to compare two compilers or flag sets.

## When to Give Up

The compiler will never produce code as good as a hand-tuned expert-written assembly kernel for a specific problem on a specific microarchitecture. The question is whether the gap matters for your use case.

For 95% of code, the compiler at `-O2 -march=native` produces excellent code. For the 5% that are hot loops in tight inner kernels:

1. Write the most compiler-friendly version of the code (restrict, const, known trip counts, `-ffast-math` if allowed).
2. If still not good enough, use SIMD intrinsics (faster than assembly, still portable-ish).
3. If *still* not good enough, consider assembly — but be aware you're signing up for portability costs and maintenance burden.

The case studies in chapters 11 and 12 show this progression for real algorithms.
