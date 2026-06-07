# Compile-Time Computation

If a value can be computed at compile time, it should be. The runtime doesn't need to calculate `sqrt(2.0)` or `fibonacci(20)` every time the program starts. This article covers `constexpr`, template metaprogramming, and practical techniques for shifting work from runtime to compile time.

## `constexpr` Functions

C++11 introduced `constexpr` functions that can be evaluated at compile time when their arguments are compile-time constants:

```rust
const fn fibonacci(n: i32) -> i32 {
    if n <= 1 { n } else { fibonacci(n - 1) + fibonacci(n - 2) }
}

const FIB20: i32 = fibonacci(20);  // Computed at compile time — zero runtime cost
let n = 15; // runtime-known value
let fib_runtime = fibonacci(n);  // Computed at runtime (argument not const)
```

A `constexpr` function must:
- Have a body consisting of a single return statement (C++11), or be more flexible (C++14+: loops, local variables, conditionals).
- Only call other `constexpr` functions.
- Only use compile-time-known arguments for compile-time evaluation.

## Compile-Time Lookup Tables

One of the most practical uses of `constexpr` is generating lookup tables:

```rust
const fn make_popcount_table() -> [i32; 256] {
    let mut table = [0i32; 256];
    let mut i = 1;
    while i < 256 {
        table[i] = table[i >> 1] + (i & 1);
        i += 1;
    }
    table
}

const POPCOUNT_TABLE: [i32; 256] = make_popcount_table();

fn popcount(x: u32) -> i32 {
    POPCOUNT_TABLE[(x & 0xFF) as usize] +
    POPCOUNT_TABLE[((x >> 8) & 0xFF) as usize] +
    POPCOUNT_TABLE[((x >> 16) & 0xFF) as usize] +
    POPCOUNT_TABLE[((x >> 24) & 0xFF) as usize]
}
```

The entire 256-entry table is compiled into the binary as initialized data. No runtime initialization, no static initialization order fiasco.

This technique is used throughout embedded systems and performance-critical code. Any table that is computationally expensive to generate but can be computed at compile time should be.

## Compile-Time Dispatch

`if constexpr` (C++17) enables conditional compilation without preprocessor macros:

```rust
trait Process {
    fn process(self) -> Self;
}

impl Process for i32 {
    fn process(self) -> i32 { self * 2 }
}

impl Process for f64 {
    fn process(self) -> f64 { self * 2.0 }
}
// Unsupported types: compile error (equivalent to static_assert)
```

Unlike a regular `if`, the discarded branches don't even need to compile for the given type. This enables writing generic code that specializes at compile time without runtime overhead.

## Template Metaprogramming

Before `constexpr` was powerful enough, compile-time computation was done with templates:

```rust
// Rust const fn (preferred over type-level metaprogramming):
const fn fibonacci(n: u32) -> u32 {
    if n <= 1 { n } else { fibonacci(n - 1) + fibonacci(n - 2) }
}

const FIB20: u32 = fibonacci(20);  // Compile-time
```

This is harder to read and write than `constexpr` functions. Prefer `constexpr` for new code; use templates only when you need SFINAE or type-level computation.

## Practical Examples

**CRC table generation**:
```rust
const fn make_crc32_table() -> [u32; 256] {
    let mut table = [0u32; 256];
    let mut i = 0;
    while i < 256 {
        let mut crc = i as u32;
        let mut j = 0;
        while j < 8 {
            crc = (crc >> 1) ^ if (crc & 1) != 0 { 0xEDB88320 } else { 0 };
            j += 1;
        }
        table[i] = crc;
        i += 1;
    }
    table
}
```

**Sine/cosine tables** for audio oscillators or embedded DSP:
```rust
// Note: f32::sin() is not const fn in stable Rust (same limitation as C++).
// Use build-time code generation (shown below) for this case.
const fn make_sin_table() -> [f32; 1024] {
    let mut table = [0.0f32; 1024];
    let mut i = 0;
    while i < 1024 {
        table[i] = (2.0 * std::f32::consts::PI * i as f32 / 1024.0).sin();
        i += 1;
    }
    table
}
```

**Compile-time string hashing** (for switch-on-string):
```rust
const fn hash(s: &[u8], h: u32) -> u32 {
    if s.is_empty() {
        h
    } else {
        hash(&s[1..], h.wrapping_mul(31).wrapping_add(s[0] as u32))
    }
}

let h = hash(input.as_bytes(), 0);
if h == hash(b"open", 0) {
    /* ... */
} else if h == hash(b"close", 0) {
    /* ... */
}
```

## Limitations

`constexpr` cannot:
- Allocate dynamic memory (until C++20, and even then, allocations must be deallocated at compile time).
- Use `reinterpret_cast` or access the representation of an object (no type punning at compile time — though `std::bit_cast` works in C++20).
- Call non-`constexpr` functions (including most of `<cmath>` — `std::sin`, `std::log`, etc. are not `constexpr` until C++26).
- Access file I/O, network, or any OS service.

For floating-point math that is not `constexpr`, generate the tables at build time using a separate program (a "build-time code generator") and include the generated array in your source.

## Build-Time Computation via Code Generation

For complex computations beyond what `constexpr` can express:

```bash
# Generate a file during the build
python3 generate_tables.py > tables.inc
```

```rust
// Include the generated file at build time
const OPTIMIZATION_TABLE: [i32; 1024] = {
    include!("tables.inc")
};
```

This is how many performance-critical libraries handle initialization — the build system runs a code generator that produces C source files, which are then compiled normally. The Makefile ensures the generated file is rebuilt when the generator changes.

This approach is more flexible than `constexpr` (you can use any language for generation, access files, run benchmarks to find optimal parameters) but adds build complexity.
