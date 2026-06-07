# Array of Structures vs. Structure of Arrays

The AoS vs. SoA choice is one of the most impactful data layout decisions you can make. The difference can be 3–5× in memory-bound code. This article measures the effect and explains when each layout wins.

## The Two Layouts

```rust
// Array of Structures (AoS)
struct Point3D { x: f32, y: f32, z: f32 }
// let points: Vec<Point3D> = Vec::with_capacity(N);

// Structure of Arrays (SoA)
struct Points3D { x: Vec<f32>, y: Vec<f32>, z: Vec<f32> }
// Where x[i], y[i], z[i] correspond to point i
```

Memory layout for AoS (N=4):
```
[x0][y0][z0][x1][y1][z1][x2][y2][z2][x3][y3][z3]
```

Memory layout for SoA:
```
[x0][x1][x2][x3] ... [y0][y1][y2][y3] ... [z0][z1][z2][z3]
```

## The Experiment

Three workloads:

### Workload 1: Access All Fields

```rust
// Compute squared distance from origin
for i in 0..n {
    sq += points[i].x * points[i].x +
          points[i].y * points[i].y +
          points[i].z * points[i].z;
}
```

**Winner**: AoS (slightly). All three coordinates of each point are in the same cache line (or two adjacent lines). The CPU loads one line and gets all three values. SoA would need to load from three separate arrays, potentially causing more cache misses.

### Workload 2: Access One Field

```rust
// Sum x-coordinates only
for i in 0..n {
    sum += points[i].x;
}
```

**Winner**: SoA (dramatically). The loop streams through `x[]` — every cache line contains 16 useful floats. With AoS, every cache line contains only 5–6 useful floats (x, y, z, x, y, z — but we only need x). 2/3 of the memory bandwidth is wasted on unused y and z values.

Measured speedup on Zen 2: SoA is **~3× faster** when scanning one field.

### Workload 3: Mixed Access

Some loops access x and y but not z. Some access all three. The tradeoff becomes complex.

## RAM Row Buffer Effects

An additional factor: DRAM row buffers. When accessing SoA, the `x` array is contiguous → all accesses hit the same DRAM row → fast. The `y` array is also contiguous → fast. But if `x`, `y`, and `z` are allocated separately, they may be in different DRAM banks → the row buffer must be switched between arrays → slower.

**Padded AoS**: A hybrid layout that stores AoS but pads each structure to a power-of-two (e.g., 64 bytes):

```rust
#[repr(C, align(64))]
struct Point3D_Padded { x: f32, y: f32, z: f32, pad: [u8; 52] }
```

This wastes memory but avoids associativity conflicts and DRAM row thrashing. For some access patterns, padded AoS outperforms both AoS and SoA.

## Huge Pages Interaction

With 4 KB pages and SoA, the `x`, `y`, and `z` arrays are each in separate virtual address regions. If each array is large, they use many TLB entries. With 2 MB huge pages, a single TLB entry covers 512× more data, reducing TLB pressure for SoA.

AoS naturally groups x, y, z for each point together, which fits in a single TLB entry more often — but only for small N. For large N, the TLB still needs many entries because the points are spread across many pages.

## When to Use Each Layout

**Use AoS when**:
- Each access touches most or all fields of a structure.
- The code is object-oriented (each object is self-contained).
- You need to pass individual elements to functions frequently.
- The working set is small enough to fit in L1/L2.

**Use SoA when**:
- Hot loops access only a subset of fields.
- You're doing SIMD operations on one field (SoA is naturally vectorizable — contiguous floats).
- Memory bandwidth is the bottleneck.
- You're doing columnar analytics (filter on field x, aggregate field y).

**Use padded AoS when**:
- You need the AoS API but want to avoid cache associativity problems.
- Memory is plentiful and you want predictable performance.

The data structures chapter (`data-structures/segment-trees.md`) applies these principles to segment tree implementations, where the choice of layout affects performance by 2–3×.
