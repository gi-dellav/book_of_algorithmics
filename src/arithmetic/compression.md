# Data Compression

Compression reduces the number of bits needed to represent data. In performance work, compression is about trading CPU cycles for memory bandwidth: if data is compressed, less data moves across the memory bus, and that saving often outweighs the cost of compression/decompression. This article covers lightweight compression techniques that are fast enough for hot paths.

## When Compression Wins

A Zen 2 core can do ~4 integer operations per cycle at 2 GHz — about 8 billion operations per second. Memory bandwidth is ~25 GB/s (single channel DDR4). If we can save 4 bytes of memory traffic by doing 16 integer operations, we break even. 16 integer operations can decompress quite a lot — run-length encoding, delta decoding, or varint parsing all fall within this budget.

The formula: **compression is beneficial when `ops_per_byte_saved < cpu_throughput / memory_bandwidth`**.

For Zen 2: 8e9 ops/s / 25e9 bytes/s = 0.32 ops/byte. If saving 1 byte of memory traffic costs less than 0.32 integer operations, compression is net positive. Most lightweight compression schemes are well within this budget.

## Run-Length Encoding (RLE)

Replace consecutive identical values with a count and a value:

```
Input:  AAAABBBCCD
Output: A4 B3 C2 D1
```

```c
int rle_encode(const char *src, int n, char *dst) {
    int out = 0;
    for (int i = 0; i < n; ) {
        char c = src[i];
        int count = 1;
        while (i + count < n && src[i + count] == c && count < 255)
            count++;
        dst[out++] = c;
        dst[out++] = count;
        i += count;
    }
    return out;
}
```

RLE works well for: sparse bitmaps, black-and-white images, mask arrays with long runs of identical values. It's useless for: random data, floats from a sensor (no two values identical), text (rarely has runs longer than 2).

Throughput: encoding is ~2–3 cycles per byte; decoding is ~1 cycle per byte. Net win if the data compresses to less than ~80% of original size.

## Delta Coding

Store the difference between consecutive values, not the values themselves. If values change slowly, differences are small and require fewer bits:

```
Input:  [1000, 1003, 1008, 1007, 1010]
Deltas: [1000, +3, +5, -1, +3]
```

```c
void delta_encode(const int *src, int n, int *dst) {
    dst[0] = src[0];
    for (int i = 1; i < n; i++)
        dst[i] = src[i] - src[i-1];
}
```

Combine with varint encoding: small deltas → few bytes. Used in time-series databases, audio codecs, and video compression (motion vectors are deltas).

## Varint Encoding (Variable-Length Integers)

Most integers in real data are small. A 4-byte `int` can store values up to 2³¹, but if most values are under 128, the top 3 bytes are always zero. Varint encoding stores small numbers in fewer bytes:

```c
// Encode uint32 as varint (up to 5 bytes)
int varint_encode(uint32_t x, uint8_t *dst) {
    int pos = 0;
    while (x >= 0x80) {
        dst[pos++] = (x & 0x7F) | 0x80;  // Lower 7 bits, continuation bit set
        x >>= 7;
    }
    dst[pos++] = x & 0x7F;  // Final byte, continuation bit clear
    return pos;
}

uint32_t varint_decode(const uint8_t *src, int *consumed) {
    uint32_t x = 0;
    int shift = 0;
    int pos = 0;
    while (1) {
        uint8_t b = src[pos++];
        x |= (uint32_t)(b & 0x7F) << shift;
        if (!(b & 0x80)) break;
        shift += 7;
    }
    *consumed = pos;
    return x;
}
```

Protobuf, Cap'n Proto, and FlatBuffers all use varint encoding. Throughput: ~1 byte per 2 cycles for decoding. Very fast.

## Delta-of-Delta + Varint

A powerful combination: compute deltas, then deltas of deltas (second differences), then varint-encode. A linearly-increasing sequence becomes constant second differences:

```
Original:  [100, 200, 300, 400, 500]
First delta:  [100, +100, +100, +100, +100]
Second delta: [100, 0, 0, 0, 0]
```

Now the entire sequence (after the first two values) is zeros — 1 byte each. Used in Facebook's Gorilla time-series database.

## SIMD-Accelerated Compression

SIMD can process 16–64 bytes at once, making compression and decompression dramatically faster.

**SIMD bit-packing**: For values with known small range, pack 32 values into the fewest bits possible:
```c
// Pack 32 4-bit values into 16 bytes
__m128i pack_4bit(const uint32_t *src) {
    __m128i v0 = _mm_loadu_si128((__m128i*)src);      // values 0-3
    __m128i v1 = _mm_loadu_si128((__m128i*)(src+4));  // values 4-7
    // ... pack with shifts and ORs ...
}
```

**SIMD RLE**: Detect runs of identical 32-bit values using `_mm256_cmpeq_epi32` between adjacent elements. The movemask gives run boundaries directly.

**SIMD memcpy**: For bulk compression where the compacted size isn't known in advance, SIMD gather/scatter or masked stores can write variable-length elements.

## Entropy Coding: Huffman

Huffman coding assigns shorter codes to more frequent symbols. A prefix code ensures no code is a prefix of another:

```
Symbol frequencies: A=50%, B=25%, C=15%, D=10%
Huffman tree:
   / \
  A   /\
     B /\
      C  D
Codes: A=0, B=10, C=110, D=111
```

Build the tree once (from known or sampled frequencies), then encode/decode using lookup tables:

```c
// Decoding with a 256-entry lookup table (for codes up to 8 bits)
struct { uint8_t symbol; uint8_t length; } decode_table[256];
// Populated so that for any 8-bit prefix, we know the first symbol and its length
```

SIMD Huffman decoding is an active research area. Modern approaches use `pext` (BMI2) to extract variable-length fields, or AVX-512 `vpermb` for table-driven decoding of multiple symbols per cycle.

## Asymmetric Numeral Systems (ANS)

ANS is the state-of-the-art in entropy coding, used in zstd, Facebook Zstandard, and Apple LZFSE. It achieves compression ratios within 0.1% of the Shannon limit while being much faster than arithmetic coding.

The core idea: represent the compressed stream as a single integer, with information theoretic packing of symbols. For performance, the integer is broken into chunks and processed with SIMD.

zstd level 1 achieves ~2–3× compression at ~500 MB/s on a single core. This is fast enough to be used in transparent filesystem compression (btrfs, ZFS) and network compression.

## Practical Compression Heuristics

1. **Measure before compressing.** If your data is already random, compression makes it larger.
2. **Use byte-aligned lightweight schemes.** RLE, delta, and varint are fast enough for inner loops.
3. **Consider compression level.** zstd level 1 is 10× faster than level 19 and gives 80% of the compression ratio.
4. **Batch compression.** Compressing lots of small buffers separately is inefficient; combine them into a single stream.
5. **Prefetch compressed data.** If decompression is memory-bandwidth-bound, prefetch the next block while decompressing the current one.
6. **The CPU is faster than the memory bus.** Compression that reduces memory traffic by 30% and costs 0.5 cycles/byte is a net win at 25 GB/s bandwidth. Almost all lightweight compression clears this bar.
