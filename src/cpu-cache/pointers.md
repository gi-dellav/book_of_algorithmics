# Pointers and Alternatives

Pointers are expensive. A 64-bit pointer occupies 8 bytes of storage, and dereferencing it causes a dependent load that kills memory-level parallelism. This article explores alternatives: indices, relative pointers, and packed formats that reduce memory overhead and improve cache utilization.

## Pointers vs. Indices

```rust
// Using pointers
struct Node { value: i32, next: *const Node }  // 16 bytes per node

// Using indices
struct Node { value: i32, next_idx: i32 }  // 8 bytes per node
// let mut nodes: Vec<Node> = Vec::with_capacity(MAX_NODES);
```

The index version:
- Halves the storage per node (4-byte index vs. 8-byte pointer).
- Uses a dense array rather than scattered heap allocations (better cache locality).
- Enables bounds-checked access (security).
- Allows 32-bit indices even on 64-bit systems.

Benchmark (Zen 2, L1-resident linked list traversal):
- Pointer chasing: ~3 ns per node (pointer dereference + next load).
- Index chasing: ~2 ns per node (array access to `nodes[idx].next_idx`).

The difference: with indices, the entire array is contiguous, so the hardware can compute the next address (`nodes + next_idx * sizeof(Node)`) without a memory load for the address. With pointers, the `next` pointer must be loaded before the address of the next node is known — a true data dependency.

## 32-bit vs. 64-bit Pointers

On x86-64, the x32 ABI uses 32-bit pointers with the 64-bit instruction set. This gives the advantages of 64-bit registers and addressing but with 4-byte pointers. Memory savings: 25–40% for pointer-heavy data structures. Performance improvement: 5–15% for cache-resident workloads (more objects fit in the same cache).

Drawback: limited address space (4 GB), not compatible with standard libraries built for 64-bit. The x32 ABI is niche but demonstrates a principle: pointer width is a design choice, and for data structures with trillions of pointers, 4-byte indices outperform 8-byte pointers.

## 24-Bit Pointers

For truly memory-constrained applications, pack a 24-bit index and an 8-bit flag into one 32-bit word:

```rust
struct PackedNode {
    value: i32,
    packed: u32,  // next_idx in low 24 bits, flag in high 8 bits
}

impl PackedNode {
    fn next_idx(&self) -> u32 { self.packed & 0x00FF_FFFF }
    fn flag(&self) -> u8 { (self.packed >> 24) as u8 }
    fn set_next_idx(&mut self, idx: u32) {
        self.packed = (self.packed & 0xFF00_0000) | (idx & 0x00FF_FFFF);
    }
    fn set_flag(&mut self, flag: u8) {
        self.packed = (self.packed & 0x00FF_FFFF) | ((flag as u32) << 24);
    }
}
```

The compiler generates bitfield extract/insert instructions (`bextr`, `bzhi` on BMI2). Extraction is 1 cycle; packing requires a read-modify-write of the 32-bit word. Use when the 2× memory reduction justifies the extra instruction.

## Pointer Tagging

Steal bits from pointers to store metadata. Since x86-64 uses only the low 48 bits of a 64-bit pointer (the high 16 bits must be sign-extensions of bit 47), you can use the high 16 bits for tags:

```rust
let tagged = (ptr as usize) | (tag << 48);
// Later:
let ptr = ((tagged << 16) as isize >> 16) as *mut std::ffi::c_void;  // Sign-extend from bit 47
let tag = tagged >> 48;
```

Or use the low 3 bits (always zero for 8-byte-aligned pointers):
```rust
let tagged = (ptr as usize) | type_tag;  // type_tag in {0, 1, ..., 7}
// Later:
let ptr = (tagged & !7) as *mut std::ffi::c_void;
let tag = tagged & 7;
```

Pointer tagging is used in JavaScript engines (V8: tagged pointers for small integers/SMI, doubles, objects), Objective-C runtime ("tagged pointers" for small objects like NSNumber), and Linux kernel (`rb_root` uses low bits of root pointer for color).

## Relative Pointers

Store the offset between the pointer and the pointed-to object, rather than the absolute address:

```rust
struct RelNode {
    value: i32,
    next_offset: i32,  // Offset from this node to the next
}

impl RelNode {
    fn next(&self) -> *const RelNode {
        // SAFETY: caller must ensure the computed pointer is valid
        unsafe { (self as *const Self as *const u8).offset(self.next_offset as isize) as *const RelNode }
    }
}
```

Then `next = (RelNode *)((char *)this + next_offset)`. Relative pointers enable position-independent data structures — they can be `mmap`'d at any address and work without relocation. Used in shared memory IPC, persistent data structures, and some file formats (FlatBuffers, Cap'n Proto).

The offset is typically 32 bits (4 GB range), sufficient for most in-memory data structures. If the offset fits in 16 bits (64 KB range), even more compact.

## Skip Lists with Indices

A skip list can use array indices instead of pointers:

```rust
struct SkipNode {
    key: i32,
    value: i32,
    next: [i32; SKIP_LEVELS],  // Indices (i32) instead of pointers
}
// let mut pool: Vec<SkipNode> = Vec::with_capacity(MAX_NODES);
```

The memory savings (4 bytes vs. 8 bytes per forward pointer, per level) accumulate quickly. For a skip list with 16 levels and 1 million nodes:
- Pointers: 1M × 16 × 8 = 128 MB for forward pointers alone.
- Indices: 1M × 16 × 4 = 64 MB.

The 64 MB difference can mean the difference between fitting in L3 and spilling to RAM — a 5× performance gap.

## When Pointers Are Better

Pointers have advantages:
- **Universality**: Any memory address can be referenced.
- **No base array**: You don't need to carry around a pointer to the backing array.
- **Standard tooling**: Debuggers, profilers, and memory analyzers understand pointers natively.
- **No bounds checks**: Faster (but less safe).

For performance-critical data structures where the backing store is known and bounded, indices are almost always better. For general-purpose code, pointers are simpler and more flexible. The data structures chapter applies both approaches, choosing the right tool for each case study.
