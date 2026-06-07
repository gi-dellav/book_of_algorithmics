# Contracts and Undefined Behavior

Undefined behavior (UB) is the dark matter of C and C++. It's invisible in correct programs, but its gravitational pull shapes everything the compiler does. Understanding UB — what it is, why it exists, and how compilers exploit it — is essential for writing code that is both correct and fast.

## What Is Undefined Behavior?

The C and C++ standards explicitly decline to define the behavior of certain constructs. When a program executes an operation with undefined behavior, the standard imposes no requirements. The program might crash, produce wrong results, appear to work correctly, or format your hard drive. The compiler is under no obligation to warn you or produce consistent results.

Examples of UB:
- Signed integer overflow (`INT_MAX + 1`).
- Dereferencing a null pointer.
- Accessing out-of-bounds array elements.
- Using a variable before it's initialized.
- Data races (two threads accessing the same variable without synchronization, at least one writing).
- Violating strict aliasing rules.
- Division by zero.
- Infinite recursion without side effects.
- Returning from a non-void function without a value.

## Why Does UB Exist?

UB is not a mistake. It's a deliberate design choice that enables optimization.

Consider signed overflow:
```rust
fn check_overflow(x: i32) -> bool {
    x + 1 > x  // UB in C (signed overflow); Rust panics in debug, wraps in release
}
```

If signed overflow were defined (wrapping, like unsigned), `x + 1 > x` would be true except when `x == INT_MAX`. The compiler would have to generate code to check for overflow. But because signed overflow is UB, the compiler can assume it never happens and optimize `x + 1 > x` to `true`. This eliminates a branch.

More importantly, UB enables the compiler to reason about loop bounds:

```rust
for i in 0..=n { /* ... */ }  // Panics on overflow in debug, wraps in release
```

If `n == INT_MAX`, the loop would run forever (or overflow). Since signed overflow is UB, the compiler can assume `n < INT_MAX` and generate a count-toward-zero loop that's one instruction shorter.

**UB is the compiler's license to assume your program is well-behaved and optimize accordingly.**

## UB That Bites

The most dangerous UB is the kind that *appears* to work at `-O0` but breaks at `-O2`:

```rust
// BUG: signed overflow
fn midpoint(a: i32, b: i32) -> i32 {
    (a + b) / 2  // UB if a + b overflows (panics in debug)
}

// FIX:
fn midpoint(a: i32, b: i32) -> i32 {
    a + (b - a) / 2  // Safe
}
```

At `-O0`, `(a + b) / 2` might wrap and produce a "reasonable" answer. At `-O2`, the compiler sees that `a + b` can't overflow (because it's UB), deduces that `a` and `b` must be small, and generates code that fails for large values.

Other common traps:

```rust
// BUG: Uninitialized variable — Rust compiler rejects this at compile time
let x: i32;
if condition { x = 5; }
// x may be used uninitialized if condition is false — error[E0381]!

// BUG: Out-of-bounds access
let mut a = [0i32; 10];
a[10] = 0;  // Rust: panic at runtime, not UB

// BUG: Use-after-free — prevented by ownership
let p = Box::new(42);
drop(p);
// *p = 42;  // Does not compile; ownership prevents use-after-free

// BUG: Violating strict aliasing — use f32::to_bits() or mem::transmute
let f = 1.0f32;
let bits: u32 = f.to_bits();  // Safe, well-defined
```

## Assumptions and Unreachable

You can explicitly communicate invariants to the compiler:

```rust
// Tell the compiler 'n' is a multiple of 8
unsafe fn process(a: &mut [f32]) {
    let n = a.len();
    if n % 8 != 0 { std::hint::unreachable_unchecked(); }
    for i in 0..n {
        a[i] *= 2.0;
    }
}
```

`__builtin_unreachable()` tells the compiler "this code path is never taken." The compiler eliminates the branch and assumes `n % 8 == 0` for all subsequent code. If the assumption is wrong, the behavior is undefined (typically a crash or incorrect result).

Similarly, `__builtin_assume(condition)` (Clang) tells the compiler to assume a condition holds without generating a check.

## Contracts in C++

C++20 introduced `[[likely]]` and `[[unlikely]]` for branch hints. C++23 adds `[[assume(expr)]]` for portable assumptions. Future standards may add full contract programming support (preconditions, postconditions, invariants).

For now, the most portable way to express invariants is:

```rust
macro_rules! assume {
    ($cond:expr) => {
        if !$cond { unsafe { std::hint::unreachable_unchecked() } }
    };
}
```

Use it sparingly. Every assumption is a potential bug if violated.

## Sanitizer-Driven Development

The standard workflow for UB-safe optimization:

1. Write the code clearly and correctly.
2. Test with `-fsanitize=undefined,address` enabled.
3. Fix all sanitizer warnings.
4. Add `restrict`, `const`, and assumptions where justified by program logic.
5. Enable optimizations and measure.

Don't add assumptions or restrict to "fix" performance problems you haven't measured. Sanitizers are your safety net — use them.

## The Aliasing Contract

C's strict aliasing rule says you may only access an object through an lvalue of a compatible type. The exceptions: `char*`, `unsigned char*`, and `std::byte*` can alias anything.

```rust
// In Rust, strict aliasing is enforced by the borrow checker.
// Safe type-punning:

let f = 1.0f32;

// Use to_bits() for safe bitwise reinterpretation:
let bits: u32 = f.to_bits();

// Or transmute (unsafe, well-defined for same-size types):
let bits: u32 = unsafe { std::mem::transmute(f) };

// Byte-level access is always allowed:
let bytes: [u8; 4] = f.to_le_bytes();
let byte0 = bytes[0];  // OK

// Preferred: use to_bits() / from_bits()
let f = 1.0f32;
let result: u32 = f.to_bits();  // Safe, no pointer casting needed
```

Strict aliasing enables the compiler to reorder loads and stores. If it knows a `float*` and an `int*` can't alias, it can keep the float value in a register across writes through the int pointer. Violating strict aliasing leads to "impossible" bugs where a value changes apparently at random.

The fix: use `memcpy` for type-punning (the compiler optimizes it away), or use `union` (defined in C, implementation-defined in C++, may not be portable), or use `std::bit_cast` (C++20). Or better yet, avoid type-punning — most uses are unnecessary.
