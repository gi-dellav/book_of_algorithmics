# Packing Integers Tighter

Databases store billions of integers. Search engines store trillions of document IDs. Columnar analytics stores compress timestamps, counters, and identifiers. The difference between a 4-byte `u32` and a 1-byte compressed integer is a 4× reduction in storage, disk I/O, and network bandwidth. For in-memory data, it's the difference between fitting in cache and spilling to RAM.

Integer compression algorithms exploit the fact that real-world integers are not uniformly distributed. Document IDs in an inverted index are sorted and have small gaps. Timestamps in a log are monotonic with occasional bursts. Floating-point measurements have limited precision. Each pattern calls for a different compression scheme.

## What This Chapter Covers

1. **Variable-Byte and Delta Coding** — The simplest compression: use fewer bytes for smaller numbers. Delta + varint is the baseline that everything else competes against.
2. **Bit-Packing and SIMD** — Pack b-bit integers into 32- or 64-bit words. SIMD-accelerated unpacking with `_pext_u64` and `_pdep_u64`. Simple16, PForDelta, and the TurboPack family.
3. **Elias-Fano and Quasi-Succinct** — Encode monotone sequences in near-optimal space with O(1) random access. The data structure behind most inverted indexes.
4. **Dictionary and Frame-of-Reference** — Encode blocks of 128 integers relative to a frame minimum. Combined with SIMD bit-unpacking, decode at 10+ billion integers/second.

## Recommended Reading Order

Start with Variable-Byte — it's the simplest and establishes the delta coding concept used everywhere. Then Bit-Packing, which is the workhorse of production systems. Elias-Fano is the theoretical gold standard. Frame-of-Reference ties together the practical lessons.

Cross-reference with Chapter 10 (SIMD) for the unpacking intrinsics and Chapter 8 (External Memory) for why compressed integers matter for I/O-bound systems.
