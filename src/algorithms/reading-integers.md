# Parsing Integers

Converting a string like "12345" to an integer is one of the most common operations in computing — every CSV parser, JSON parser, and compiler does it millions of times. `scanf` and `strtol` are general-purpose and slow. Daniel Lemire's fast integer parsing algorithm achieves ~35× speedup using SWAR techniques and SIMD.

## The Problem

```c
int parse_int(const char *str, int len);  // "12345" → 12345
```

Millions of calls per second in data pipelines. Every cycle matters.

## Why `strtol` Is Slow

`strtol` handles:
- Optional sign (`+` or `-`).
- Optional base prefix (`0x`, `0`, `0b`).
- Arbitrary base (2–36).
- Whitespace skipping.
- Overflow detection with proper `errno`.
- Locale-dependent digit handling.

For most use cases, we know the format: unsigned, base 10, no whitespace, no sign. We can do much better.

## Stage 1: Scalar Accumulation

```c
uint64_t parse_uint64_scalar(const char *str, int len) {
    uint64_t value = 0;
    for (int i = 0; i < len; i++) {
        value = value * 10 + (str[i] - '0');
    }
    return value;
}
```

~18 cycles per byte on Zen 2. The multiplication by 10 (`imul`) and the loop carried dependency (`value = value * 10 + ...`) are the bottlenecks.

## Stage 2: SWAR Parsing (8 Bytes at a Time)

Process 8 bytes per iteration using 64-bit integer operations:

```c
uint64_t parse_uint64_swar(const char *str, int len) {
    uint64_t value = 0;
    int i = 0;
    
    for (; i + 8 <= len; i += 8) {
        uint64_t chunk;
        memcpy(&chunk, str + i, 8);  // Load 8 bytes
        
        // Check that all bytes are digits ('0'–'9'):
        uint64_t t1 = chunk + 0x4646464646464646ull;  // Convert digit range
        uint64_t t2 = chunk - 0x3030303030303030ull;  // ASCII shift
        uint64_t t3 = (chunk | 0x3030303030303030ull) - 0x3030303030303030ull;
        if ((t1 | t2) & 0x8080808080808080ull) {
            // Non-digit byte detected — fall back to scalar
            break;
        }
        
        // Parse the 8-digit chunk into a number
        // Use multiplication and addition to combine digits
        // Example: [d0,d1,d2,d3,d4,d5,d6,d7] →
        //   (((d0*10 + d1)*10 + d2)*10 + ... )*10 + d7
        //   But calculated with SIMD-like techniques
        ...
        value = value * 100000000 + chunk_value;
    }
    
    // Handle remaining bytes with scalar
    for (; i < len; i++)
        value = value * 10 + (str[i] - '0');
    
    return value;
}
```

The SWAR digit validation is the same zero-detection trick from `arithmetic/bit-hacks.md`. The chunk parsing uses a sequence of multiplications and additions that the compiler maps to efficient instruction sequences. Performance: ~3 cycles per byte (~6× faster).

## Stage 3: AVX-512 Parsing

AVX-512 can process 32 bytes at once:

```c
__m512i parse_32_digits(const char *str) {
    __m512i chunk = _mm512_loadu_si512((__m512i*)str);
    
    // Convert ASCII to digits: subtract '0' (0x30)
    __m512i digits = _mm512_sub_epi8(chunk, _mm512_set1_epi8('0'));
    
    // Validate: ensure all bytes are < 10 (i.e., they were digits)
    __mmask64 mask = _mm512_cmple_epu8_mask(digits, _mm512_set1_epi8(9));
    if (mask != ~0ull) {
        // Handle non-digit bytes
    }
    
    // Parse digits into a 64-bit integer
    // Use multiply-add to combine: (...(d0*10+d1)*10+...)...
    // For 32 digits, this requires big-integer arithmetic (exceeds 64 bits)
    // Parse in smaller groups (8 digits → 32-bit, then combine)
}
```

AVX-512 also provides `_mm512_maskz_loadu_epi8` for masked loads (handling strings shorter than 32 bytes without over-reading).

Performance: ~1 cycle per byte (~18× faster than scalar). Most of the speedup comes from processing 32 bytes per iteration and eliminating the loop overhead.

## Stage 4: The Full Implementation

Combining all techniques (Lemire's algorithm, circa 2020):

1. **SWAR digit validation**: Check 8 bytes at a time using arithmetic operations to detect non-digit bytes.
2. **8-digit parsing**: Parse an 8-digit chunk into a 32-bit integer using a specialized multiplication sequence.
3. **Accumulation**: `value = value * 100000000 + chunk_value` — the multiplication by 10^8 is a constant propagation the compiler optimizes well.
4. **AVX-512**: For machines with AVX-512, use 32-byte loads and the mask registers for validation.
5. **Overflow detection**: The final multiplication `value * 100000000` can overflow — check before multiplying.

```c
bool parse_uint64_fast(const char *str, int len, uint64_t *out) {
    uint64_t value = 0;
    int i = 0;
    
    while (i + 8 <= len) {
        uint64_t chunk;
        memcpy(&chunk, str + i, 8);
        
        // Validate digits
        if (!is_8_digits(chunk)) return false;
        
        // Parse 8 digits
        uint32_t chunk_val = parse_8_digits(chunk);
        
        // Check overflow before accumulating
        if (value > UINT64_MAX / 100000000) return false;
        value = value * 100000000 + chunk_val;
        i += 8;
    }
    
    // Parse final chunk (1–7 digits)
    while (i < len) {
        if (str[i] < '0' || str[i] > '9') return false;
        if (value > (UINT64_MAX - (str[i] - '0')) / 10) return false;
        value = value * 10 + (str[i] - '0');
        i++;
    }
    
    *out = value;
    return true;
}
```

Performance: ~1.5 cycles per byte (scalar + SWAR), ~0.5 cycles per byte (AVX-512). ~35× faster than `strtoull` on Zen 2 for 8–16 digit numbers.

## Applications

Fast integer parsing is critical for:
- **JSON/CSV parsing**: `simdjson` (Lemire) uses these techniques to parse JSON at 3+ GB/s.
- **Compiler lexers**: Converting numeric literals to integers.
- **Log processing**: Extracting numeric fields from log lines.
- **Network protocol parsers**: Reading integer fields from wire formats.

The technique generalizes to parsing floats (manually reconstruct the IEEE 754 representation) and hexadecimal numbers (using bit operations instead of multiplication by powers of 16).
