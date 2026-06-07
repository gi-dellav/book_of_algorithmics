# Compile-Time Computation

If a value can be computed at compile time, it should be. The runtime doesn't need to calculate `sqrt(2.0)` or `fibonacci(20)` every time the program starts. This article covers `constexpr`, template metaprogramming, and practical techniques for shifting work from runtime to compile time.

## `constexpr` Functions

C++11 introduced `constexpr` functions that can be evaluated at compile time when their arguments are compile-time constants:

```cpp
constexpr int fibonacci(int n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

constexpr int fib20 = fibonacci(20);  // Computed at compile time — zero runtime cost
int fib_runtime = fibonacci(rand());  // Computed at runtime (argument not constexpr)
```

A `constexpr` function must:
- Have a body consisting of a single return statement (C++11), or be more flexible (C++14+: loops, local variables, conditionals).
- Only call other `constexpr` functions.
- Only use compile-time-known arguments for compile-time evaluation.

## Compile-Time Lookup Tables

One of the most practical uses of `constexpr` is generating lookup tables:

```cpp
constexpr std::array<int, 256> make_popcount_table() {
    std::array<int, 256> table{};
    for (int i = 0; i < 256; i++)
        table[i] = table[i >> 1] + (i & 1);
    return table;
}

constexpr auto popcount_table = make_popcount_table();

// Usage:
int popcount(uint32_t x) {
    return popcount_table[x & 0xFF] +
           popcount_table[(x >> 8) & 0xFF] +
           popcount_table[(x >> 16) & 0xFF] +
           popcount_table[(x >> 24) & 0xFF];
}
```

The entire 256-entry table is compiled into the binary as initialized data. No runtime initialization, no static initialization order fiasco.

This technique is used throughout embedded systems and performance-critical code. Any table that is computationally expensive to generate but can be computed at compile time should be.

## Compile-Time Dispatch

`if constexpr` (C++17) enables conditional compilation without preprocessor macros:

```cpp
template<typename T>
auto process(const T &value) {
    if constexpr (std::is_integral_v<T>) {
        return value * 2;  // Only compiled for integer types
    } else if constexpr (std::is_floating_point_v<T>) {
        return value * 2.0;  // Only compiled for FP types
    } else {
        static_assert(sizeof(T) == 0, "Unsupported type");
    }
}
```

Unlike a regular `if`, the discarded branches don't even need to compile for the given type. This enables writing generic code that specializes at compile time without runtime overhead.

## Template Metaprogramming

Before `constexpr` was powerful enough, compile-time computation was done with templates:

```cpp
template<int N>
struct Fibonacci {
    static constexpr int value = Fibonacci<N-1>::value + Fibonacci<N-2>::value;
};

template<>
struct Fibonacci<0> { static constexpr int value = 0; };

template<>
struct Fibonacci<1> { static constexpr int value = 1; };

constexpr int fib20 = Fibonacci<20>::value;  // Compile-time
```

This is harder to read and write than `constexpr` functions. Prefer `constexpr` for new code; use templates only when you need SFINAE or type-level computation.

## Practical Examples

**CRC table generation**:
```cpp
constexpr std::array<uint32_t, 256> make_crc32_table() {
    std::array<uint32_t, 256> table{};
    for (int i = 0; i < 256; i++) {
        uint32_t crc = i;
        for (int j = 0; j < 8; j++)
            crc = (crc >> 1) ^ ((crc & 1) ? 0xEDB88320 : 0);
        table[i] = crc;
    }
    return table;
}
```

**Sine/cosine tables** for audio oscillators or embedded DSP:
```cpp
constexpr std::array<float, 1024> make_sin_table() {
    std::array<float, 1024> table{};
    for (int i = 0; i < 1024; i++)
        table[i] = std::sin(2.0f * M_PI * i / 1024.0f);
    return table;
}
```

**Compile-time string hashing** (for switch-on-string):
```cpp
constexpr uint32_t hash(const char *str, uint32_t h = 0) {
    return *str == 0 ? h : hash(str + 1, (h * 31) + *str);
}

switch (hash(input)) {
    case hash("open"):  /* ... */ break;
    case hash("close"): /* ... */ break;
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

```c
// Include the generated file
const int optimization_table[1024] = {
#include "tables.inc"
};
```

This is how many performance-critical libraries handle initialization — the build system runs a code generator that produces C source files, which are then compiled normally. The Makefile ensures the generated file is rebuilt when the generator changes.

This approach is more flexible than `constexpr` (you can use any language for generation, access files, run benchmarks to find optimal parameters) but adds build complexity.
