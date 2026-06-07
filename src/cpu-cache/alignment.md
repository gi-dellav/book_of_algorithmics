# Alignment and Padding

How data is laid out in memory — its alignment, the order of structure fields, and the padding between them — affects both cache utilization and instruction efficiency.

## Aligned Allocation

x86-64 allows unaligned memory access (you can load a 64-bit value from any address). But unaligned access has costs:

- **Cache line straddling**: A load that crosses a 64-byte boundary requires accessing two cache lines instead of one. In the worst case, it's two cache misses.
- **SIMD restrictions**: Most SIMD load instructions (`movdqa`, `vmovdqa`) require 16/32/64-byte alignment. The unaligned variants (`movdqu`, `vmovdqu`) are equally fast on modern hardware *when the data doesn't straddle a cache line*. When it does, they're slower.

```c
// Aligned allocation (C11)
float *a = aligned_alloc(64, n * sizeof(float));  // 64-byte aligned

// C++17
auto *a = new (std::align_val_t{64}) float[n];
```

`aligned_alloc` guarantees the pointer is a multiple of the alignment. The compiler can then use aligned SIMD loads/stores, saving one instruction per memory access (loading the address directly vs. computing the aligned base and offset).

## Structure Alignment

The compiler inserts padding between structure fields to satisfy alignment requirements:

```c
struct BadLayout {
    char  a;   // 1 byte, offset 0
    // 3 bytes padding (int needs 4-byte alignment)
    int   b;   // 4 bytes, offset 4
    char  c;   // 1 byte, offset 8
    // 7 bytes padding (double needs 8-byte alignment)
    double d;  // 8 bytes, offset 16
};
// Total: 24 bytes (only 14 bytes of actual data)
```

```c
struct GoodLayout {
    double d;  // 8 bytes, offset 0
    int   b;   // 4 bytes, offset 8
    char  a;   // 1 byte, offset 12
    char  c;   // 1 byte, offset 13
    // 2 bytes padding (to make total size a multiple of 8)
};
// Total: 16 bytes (same 14 bytes of data)
```

**Rule**: Sort fields by descending alignment (doubles/long longs first, then pointers/size_t, then ints/floats, then shorts, then chars). This minimizes total structure size, improves cache utilization (more objects per cache line), and may reduce the number of fields that must be loaded.

## Bit Fields

For flags and small integers, bit fields pack values into a single word:

```c
struct Flags {
    unsigned int is_visible : 1;
    unsigned int is_enabled : 1;
    unsigned int priority : 3;  // 3 bits → 0-7
    unsigned int type     : 4;  // 4 bits → 0-15
};
// Total: 9 bits → fits in 4 bytes (compiler pads to 32 bits)
```

Bit fields can be packed more tightly but have undefined layout across compilers. For portable packed structures, use manual bit manipulation (`flags & 0x1F`) or `std::bitset`.

## Structure Packing

`__attribute__((packed))` removes all padding, forcing fields to be byte-aligned:

```c
struct __attribute__((packed)) TightLayout {
    char a;    // 1 byte
    int b;     // 4 bytes (unaligned — slow on some architectures)
    double d;  // 8 bytes (unaligned — slow)
};
// Total: 13 bytes

struct TightLayout t;
t.b = 5;  // May generate multiple instructions for unaligned access!
```

Use packing only for network protocols, file formats, or memory-constrained embedded systems. The performance cost of unaligned access usually outweighs the memory savings.

## Cache Line Alignment for Hot Data

```c
struct alignas(64) HotData {
    int counters[8];
    // 32 bytes of data, 32 bytes of padding to fill cache line
};
```

A structure aligned to 64 bytes and filling exactly one cache line ensures:
- No false sharing with other data (the cache line contains only this structure).
- Alignment of fields within the structure can be optimized knowing the base is 64-byte aligned.
- The structure never straddles two cache lines.

## `std::hardware_destructive_interference_size` (C++17)

```cpp
struct alignas(std::hardware_destructive_interference_size) ThreadData {
    std::atomic<int> counter;
    // Padding to prevent false sharing with adjacent ThreadData objects
};
```

This constant (typically 64 on x86) is the minimum offset between two objects to ensure they don't share a cache line. Use it for per-thread data structures to avoid false sharing.
