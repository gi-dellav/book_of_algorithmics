# SIMD Data Movement

Getting data into and out of SIMD registers is often more expensive than the operations themselves. Understanding load/store alignment, broadcasts, extract/insert costs, and gather/scatter is essential for efficient SIMD code.

## Aligned vs. Unaligned Loads

```c
__m256 a = _mm256_load_ps(ptr);   // Requires 32-byte alignment (crashes if misaligned)
__m256 a = _mm256_loadu_ps(ptr);  // Handles any alignment
```

On modern CPUs (Haswell+), `loadu` is as fast as `load` **when the data is aligned at runtime** — the CPU optimizes the common case. When the data crosses a cache line boundary, `loadu` costs 1–2 extra cycles. The `load` variant still exists because:
- It's smaller encoding (no prefix byte).
- It guarantees a fault on misaligned access (useful for debugging alignment assumptions).

**Recommendation**: Use `loadu`/`storeu` by default. Align your data to 32 bytes (`alignas(32)`) or 64 bytes for AVX-512. The performance difference is negligible when aligned; correctness is easier.

## Register Aliasing

x86 SIMD registers overlay each other:
- `xmm0` = low 128 bits of `ymm0` = low 128 bits of `zmm0`.
- Writing to `xmm0` zeroes the upper 128 bits of `ymm0`.
- Writing to `ymm0` zeroes the upper 256 bits of `zmm0` (AVX-512VL).

This matters when mixing SSE and AVX instructions — switching from 256-bit to 128-bit and back causes zeroing of the upper lanes, potentially creating false dependencies. The compiler handles this with `vzeroupper` (zero upper halves of all YMM registers) at function boundaries. Don't mix intrinsics from different widths in the same function unless you understand the zeroing semantics.

## Extract and Insert: Surprisingly Slow

```c
int value = _mm_extract_epi32(v, 3);     // Extract lane 3: ~3 cycles
v = _mm_insert_epi32(v, value, 3);       // Insert into lane 3: ~3 cycles
```

Extract and insert move data between SIMD registers and scalar registers (or memory). They cross the boundary between the SIMD and integer domains, which costs 2–3 cycles on Zen 2. In a tight loop, this is devastating.

**Alternative**: Use in-register shuffles (see `shuffling.md`) to rearrange data without leaving the SIMD domain. Shuffle across lanes with `vpermilps`, `vperm2f128`, `vshufps`.

## Broadcast

Replicate a scalar across all lanes:

```c
__m256 v = _mm256_set1_ps(value);        // Broadcast from scalar
__m256 v = _mm256_broadcast_ss(&value);  // Broadcast from memory
```

Broadcast is a single instruction (`vbroadcastss`), throughput 1/cycle. It loads one value from memory and replicates it across all 8 lanes in one operation — much faster than 8 scalar loads + inserts.

## Mapping Vectors to Arrays

When loading data that is naturally SoA (Structure of Arrays), a contiguous load works perfectly:

```c
// x, y, z are separate arrays
__m256 vx = _mm256_loadu_ps(x + i);  // 8 x-values
__m256 vy = _mm256_loadu_ps(y + i);  // 8 y-values
__m256 vz = _mm256_loadu_ps(z + i);  // 8 z-values
```

When data is AoS (Array of Structures), you need a **transpose** to group same-field values into vectors:

```c
// points[i].x, points[i].y, points[i].z are interleaved
// Load 8 points → transpose to get 8 x values in one vector
__m256 vx, vy, vz;
// ... use vshufps, vperm2f128 to deinterleave ...
```

AoS-to-SoA transpose is a common SIMD pattern. The `shuffling.md` article covers it.

## Gather and Scatter

Load elements from non-contiguous addresses:

```c
__m256i indices = _mm256_setr_epi32(0, 10, 20, 30, 40, 50, 60, 70);
__m256 values = _mm256_i32gather_ps(base, indices, 4);  // 4 = scale (bytes per element)
```

`vgatherdps` loads 8 values from 8 *different* memory locations in a single instruction. The hardware does this as 8 sequential loads — it's **not** 8× faster than scalar. On Zen 2, gather throughput is ~1 element per 2 cycles → 16 cycles for 8 elements. It's useful when the access pattern is truly irregular and you want to save code size and front-end bandwidth, but it's usually not a performance win over scalar loads.

**Gather is slower than scalar on most CPUs.** Use it for convenience, not for speed. On Intel Skylake-X, gather is faster; on Zen 2/3, it's slower. Always benchmark.

Scatter (`vpscatterdd`) is similarly expensive. AVX-512's `vcompressps` is usually a better alternative for scattering: compute valid elements in a vector, then compress to contiguous memory.

## Practical Rules

1. **Align data, use unaligned loads.** `alignas(32)` on arrays, `loadu` in code. Safe and fast.
2. **Avoid extract/insert in hot loops.** Shuffle in-register instead.
3. **Use broadcast for scalar→vector.** `_mm256_set1_ps(x)` compiles to `vbroadcastss`.
4. **Benchmark gather before using it.** For most patterns, scalar loads + `_mm256_setr_ps` (or a manually-constructed vector) beats gather.
5. **Transpose AoS to SoA before the hot loop**, not inside it. One expensive pass, many cheap SIMD operations.
