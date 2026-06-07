# Situational Optimizations

Some optimizations can't be decided by the compiler at build time — they depend on runtime data, specific hardware characteristics, or domain knowledge that only the programmer possesses. This article covers the mechanisms for communicating that knowledge.

## Loop Unrolling Pragmas

The compiler's unrolling heuristic may be too aggressive or too conservative. You can control it:

```rust
// Rust: manually unroll for guaranteed control, or rely on LLVM heuristics.
// The compiler backend uses heuristics; manual unrolling gives precise control.

// Unroll 8× by hand:
for i in (0..n).step_by(8) {
    // Body repeated for i, i+1, ..., i+7 (manually expanded)
}

// Do not unroll — regular loop, compiler respects body size:
for i in 0..n {
    // Large body where I-cache matters more than loop overhead
}
```

Or with Clang/LLVM:
```rust
// Rust via LLVM: the backend applies similar heuristics.
// For explicit control, manually unroll (see above).
```

Unrolling helps when:
- The loop body is small (few instructions).
- Iteration count is known and divisible by the unroll factor.
- There's sufficient ILP in the body (otherwise you're just bloating code).

Unrolling hurts when:
- The loop body is large → I-cache pressure.
- The loop count is unpredictable → unrolled remainder handling adds branches.
- The body is latency-bound → unrolling doesn't help without multiple accumulators.

When in doubt, measure. The difference is usually small (<10%), but sometimes the compiler makes a very wrong choice.

## Inline Control

```rust
#[inline(always)]
fn hot_function() { /* ... */ }

#[inline(never)]
fn cold_function() { /* ... */ }
```

Use `always_inline` for tiny functions in hot paths where the call overhead is significant relative to the body. Use `noinline` for:
- Cold functions (error handling, initialization) — reduces I-cache pressure on the caller.
- Functions called through function pointers (inlining can't happen anyway, but the compiler may clone the function).
- Functions you want to profile individually (perf shows function-level counts).

C++17 adds:
```rust
#[inline(always)]
fn hot() { /* ... */ }

#[inline(never)]
fn cold() { /* ... */ }
```

## Branch Hints

```rust
// Nightly branch hints:
use std::intrinsics::{likely, unlikely};

if likely(!ptr.is_null()) {  // Likely
    // hot path
}

if unlikely(x == 0) {  // Equivalent to C++20 [[unlikely]]
    // cold path
}
```

These affect code layout (cold path moved out of line), not branch prediction on modern CPUs. The real benefit is reducing front-end pressure by keeping the hot path contiguous. See `architecture/layout.md` for details.

## `restrict` and Aliasing

The `restrict` keyword tells the compiler that a pointer is the *only* way to access the memory it points to during its lifetime. Two restricted pointers never alias (point to overlapping memory).

```rust
fn add_arrays(a: &[i32], b: &[i32], c: &mut [i32]) {
    // Rust's &mut guarantees no aliasing with a or b (borrow rules).
    // Equivalent to `restrict` on all three pointer parameters.
    for i in 0..c.len() {
        c[i] = a[i] + b[i];
    }
}
```

Without `restrict`, the compiler must assume `c` might alias `a` or `b`. If `c[i] = a[i] + b[i]` modifies `c[5]`, and `c` and `a` point to the same array, the next iteration's `a[i]` might read the value we just wrote. This prevents vectorization (the loop has a read-after-write dependency).

With `restrict`, the compiler assumes no aliasing. It can vectorize, reorder, and parallelize freely.

**This is the single most important C keyword for performance.** Always use `restrict` on pointer parameters that don't alias.

## Assumptions

You can communicate runtime invariants to the compiler:

```rust
// Tell the compiler that n is a multiple of 8:
unsafe { std::hint::assert_unchecked(n % 8 == 0); }

// Or mark unreachable paths:
if n % 8 != 0 { unsafe { std::hint::unreachable_unchecked(); } }

// No C++23-style [[assume]] in stable Rust yet.
```

The compiler can use these to:
- Eliminate remainder-handling code in unrolled loops.
- Prove that an index is always in bounds.
- Eliminate null checks after a pointer has been dereferenced.

**Danger**: If the assumption is wrong, the program may produce incorrect results or crash. Use only for invariants you can prove.

## Profile-Guided Optimization in Depth

PGO addresses the "compiler doesn't know runtime values" problem by measuring actual runtime behavior. It's the single most effective optimization you can enable with zero code changes.

The profile data records:
- **Edge counts**: How many times each branch direction was taken. Enables hot/cold code splitting.
- **Value profiles**: What values were observed at specific sites. Enables devirtualization ("this virtual call was always to `DerivedFoo` — inline it with a guard").
- **Function call counts**: How many times each function was called. Guides inlining decisions.

For best results:
- Use a **representative** training workload. PGO optimizes for the profile you give it; if production usage differs, PGO may be worse than no PGO.
- Collect profiles from **multiple runs** to smooth out variance.
- Update profiles when the code or usage patterns change significantly.

Clang supports **CSIRPGO** (Context-Sensitive IR-based PGO) which records the full calling context, enabling more precise optimization. This requires the `-fcs-profile-generate` flag.

## Multiversioned Functions

When a function can be optimized differently for different hardware, you can provide multiple implementations and dispatch at runtime:

```rust
// Rust: use #[target_feature] with runtime feature detection:
#[target_feature(enable = "avx2")]
unsafe fn dot_product_avx2(a: &[f32], b: &[f32]) -> f32 {
    a.iter().zip(b.iter()).map(|(x, y)| x * y).sum()
}

#[target_feature(enable = "avx512f")]
unsafe fn dot_product_avx512f(a: &[f32], b: &[f32]) -> f32 {
    a.iter().zip(b.iter()).map(|(x, y)| x * y).sum()
}

#[inline(never)]
fn dot_product_default(a: &[f32], b: &[f32]) -> f32 {
    a.iter().zip(b.iter()).map(|(x, y)| x * y).sum()
}

fn dot_product(a: &[f32], b: &[f32]) -> f32 {
    if is_x86_feature_detected!("avx512f") {
        unsafe { dot_product_avx512f(a, b) }
    } else if is_x86_feature_detected!("avx2") {
        unsafe { dot_product_avx2(a, b) }
    } else {
        dot_product_default(a, b)
    }
}
```

The compiler generates three versions: a baseline, an AVX2 version, and an AVX-512 version. The runtime linker selects the best one based on the CPU's capabilities. The programmer writes one function; the compiler handles the rest.

This is standard practice in performance-critical libraries (glibc `memcpy`, OpenBLAS kernels).

## Compiler Barriers

The compiler may reorder memory accesses or eliminate "redundant" loads/stores. When you need to prevent this (e.g., for lock-free data structures or interacting with hardware):

```rust
// Full compiler barrier (prevents reordering, no CPU instructions emitted):
std::sync::atomic::compiler_fence(std::sync::atomic::Ordering::SeqCst);
```

This doesn't emit any instruction — it tells the compiler that memory may have been modified, so all cached values in registers must be reloaded and all pending writes must be flushed. Not needed for normal code; essential for lock-free programming.
