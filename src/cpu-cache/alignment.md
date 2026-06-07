# Alignment and Padding

How data is laid out in memory — its alignment, the order of structure fields, and the padding between them — affects both cache utilization and instruction efficiency.

## Aligned Allocation

x86-64 allows unaligned memory access (you can load a 64-bit value from any address). But unaligned access has costs:

- **Cache line straddling**: A load that crosses a 64-byte boundary requires accessing two cache lines instead of one. In the worst case, it's two cache misses.
- **SIMD restrictions**: Most SIMD load instructions (`movdqa`, `vmovdqa`) require 16/32/64-byte alignment. The unaligned variants (`movdqu`, `vmovdqu`) are equally fast on modern hardware *when the data doesn't straddle a cache line*. When it does, they're slower.

```rust
// Aligned allocation
let layout = std::alloc::Layout::from_size_align(
    n * std::mem::size_of::<f32>(),
    64  // 64-byte aligned
).unwrap();
let a = unsafe { std::alloc::alloc(layout) as *mut f32 };
```

`aligned_alloc` guarantees the pointer is a multiple of the alignment. The compiler can then use aligned SIMD loads/stores, saving one instruction per memory access (loading the address directly vs. computing the aligned base and offset).

## Structure Alignment

The compiler inserts padding between structure fields to satisfy alignment requirements:

```rust
#[repr(C)]
struct BadLayout {
    a: u8,   // 1 byte, offset 0
    // 3 bytes padding (i32 needs 4-byte alignment)
    b: i32,  // 4 bytes, offset 4
    c: u8,   // 1 byte, offset 8
    // 7 bytes padding (f64 needs 8-byte alignment)
    d: f64,  // 8 bytes, offset 16
}
// Total: 24 bytes (only 14 bytes of actual data)
```

```rust
#[repr(C)]
struct GoodLayout {
    d: f64,  // 8 bytes, offset 0
    b: i32,  // 4 bytes, offset 8
    a: u8,   // 1 byte, offset 12
    c: u8,   // 1 byte, offset 13
    // 2 bytes padding (to make total size a multiple of 8)
}
// Total: 16 bytes (same 14 bytes of data)
```

**Rule**: Sort fields by descending alignment (doubles/long longs first, then pointers/size_t, then ints/floats, then shorts, then chars). This minimizes total structure size, improves cache utilization (more objects per cache line), and may reduce the number of fields that must be loaded.

## Bit Fields

For flags and small integers, bit fields pack values into a single word:

```rust
// Rust does not have bit fields. Use manual bit manipulation:
struct Flags(u32);

impl Flags {
    const IS_VISIBLE: u32 = 1 << 0;
    const IS_ENABLED: u32 = 1 << 1;
    const PRIORITY_SHIFT: u32 = 2;
    const PRIORITY_MASK: u32 = 0b111 << 2;  // 3 bits → 0-7
    const TYPE_SHIFT: u32 = 5;
    const TYPE_MASK: u32 = 0b1111 << 5;     // 4 bits → 0-15
}
// Total: 9 bits → fits in 4 bytes (u32)
```

Bit fields can be packed more tightly but have undefined layout across compilers. For portable packed structures, use manual bit manipulation (`flags & 0x1F`) or `std::bitset`.

## Structure Packing

`#[repr(packed)]` removes all padding, forcing fields to be byte-aligned:

```rust
#[repr(C, packed)]
struct TightLayout {
    a: u8,    // 1 byte
    b: i32,   // 4 bytes (unaligned — slow on some architectures)
    d: f64,   // 8 bytes (unaligned — slow)
}
// Total: 13 bytes

// Accessing packed fields may generate multiple instructions for unaligned access!
```

Use packing only for network protocols, file formats, or memory-constrained embedded systems. The performance cost of unaligned access usually outweighs the memory savings.

## Cache Line Alignment for Hot Data

```rust
#[repr(align(64))]
struct HotData {
    counters: [i32; 8],
    // 32 bytes of data, 32 bytes of padding to fill cache line
}
```

A structure aligned to 64 bytes and filling exactly one cache line ensures:
- No false sharing with other data (the cache line contains only this structure).
- Alignment of fields within the structure can be optimized knowing the base is 64-byte aligned.
- The structure never straddles two cache lines.

## `std::hardware_destructive_interference_size` (C++17)

```rust
// std::hardware_destructive_interference_size is not yet stable in Rust.
// On x86-64 the typical value is 64.
use std::sync::atomic::AtomicI32;

#[repr(align(64))]
struct ThreadData {
    counter: AtomicI32,
    // Padding to prevent false sharing with adjacent ThreadData objects
}
```

This constant (typically 64 on x86) is the minimum offset between two objects to ensure they don't share a cache line. Use it for per-thread data structures to avoid false sharing.
