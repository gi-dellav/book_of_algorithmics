# Error Correction

When data traverses a noisy channel — a wireless link, a storage medium, deep space — errors occur. Error-correcting codes add redundancy so the receiver can detect and correct errors without retransmission. This article covers the codes that are both mathematically elegant and performance-critical.

## The Basic Idea

Add k redundant bits to n data bits, creating an (n+k)-bit codeword. The redundancy allows:
- **Error detection**: The receiver can tell if errors occurred (but not correct them).
- **Error correction**: The receiver can identify and fix up to t erroneous bits.

The **Hamming distance** between two codewords is the number of bit positions where they differ. A code with minimum distance d can:
- Detect up to d−1 errors.
- Correct up to ⌊(d−1)/2⌋ errors.

## Hamming Codes

Hamming codes (Hamming, 1950) are the simplest error-correcting codes. For m parity bits, they protect up to 2^m − m − 1 data bits. The classic Hamming(7,4) code: 4 data bits + 3 parity bits = 7-bit codeword, corrects 1 error.

Parity bits are computed as XOR of specific data bits. The positions of parity bits (1, 2, 4, 8, ...) are powers of two:

```
Position: 1  2  3  4  5  6  7
Role:     P1 P2 D1 P4 D2 D3 D4

P1 = D1 ⊕ D2 ⊕ D4  (covers positions where bit 0 of index is 1)
P2 = D1 ⊕ D3 ⊕ D4  (covers positions where bit 1 of index is 1)
P4 = D2 ⊕ D3 ⊕ D4  (covers positions where bit 2 of index is 1)
```

To decode: recompute parity bits. If they differ from the received parity bits, the XOR gives the index of the erroneous bit. Flip that bit. Done.

**SIMD implementation**: Encode/decode multiple messages in parallel using SIMD bit operations. The parity computation is just XOR trees — perfect for SIMD.

## Reed-Solomon Codes

Reed-Solomon (RS) codes operate on symbols (typically 8-bit bytes) rather than bits. An RS(n, k) code takes k data symbols and produces n code symbols (n > k), able to correct up to (n−k)/2 symbol errors.

The math: view the k data symbols as coefficients of a polynomial of degree k−1 over GF(2^8). Evaluate this polynomial at n distinct points. Any k of the n evaluations suffice to reconstruct the original polynomial (via Lagrange interpolation or the Berlekamp-Massey algorithm).

RS codes are used in:
- **QR codes**: RS(26, 12) or similar, correcting up to 7 byte errors.
- **CDs/DVDs/Blu-ray**: Cross-Interleaved RS coding, correcting burst errors from scratches.
- **DSL**: RS coding on each tone.
- **Space communications**: Voyager used RS(255, 223) — correcting up to 16 byte errors per 255-byte block.

**Encoding**: `for each x_i: y_i = poly_evaluate(data_poly, x_i)`. Evaluation can be done with Horner's method, SIMD, or FFT over GF(2^8) (for n = 255).

**Decoding**: Much more complex. The Berlekamp-Massey algorithm finds the error locator polynomial, then Chien search finds the roots (error positions), then Forney's algorithm computes the error magnitudes. All over GF(2^8).

Intel ISA-L (Intel Storage Acceleration Library) provides heavily SIMD-optimized RS encoding/decoding. Throughput: multi-GB/s.

## LDPC (Low-Density Parity-Check)

LDPC codes (Gallager, 1963; revived in the 1990s) approach the Shannon capacity limit. Used in Wi-Fi (802.11n/ac/ax), 5G, DVB-S2 (satellite TV), and SSDs.

Structure: a sparse parity-check matrix H. Decoding uses belief propagation — an iterative message-passing algorithm where each bit node and check node exchanges probabilistic information about which bits are likely in error.

Performance: LDPC decoders are parallelizable (each node processes independently) and can be implemented efficiently in hardware. Software decoding is possible but CPU-intensive; hardware acceleration is common in wireless chipsets and SSD controllers.

## Practical Considerations

1. **Error correction vs. retransmission**: For low-latency links (Wi-Fi), correcting single-bit errors is faster than requesting retransmission. For bulk data (TCP), retransmission is simpler and the error rates are low enough that correction isn't needed.

2. **Software vs. hardware**: Hardware implementations (AES-NI for CRC, dedicated ECC units in memory controllers) are orders of magnitude faster. Use hardware when available.

3. **CRC for detection**: For error detection only (not correction), CRC-32 is ubiquitous (Ethernet, gzip, PNG). `_mm_crc32_u64` computes CRC-32 in hardware (~1 cycle per 8 bytes). Use it instead of software CRC when performance matters.

4. **Galois field arithmetic**: All RS and many LDPC implementations rely on fast GF(2^8) operations. The log/exp table approach from `finite.md` is standard. For SIMD, look-up in 16 registers simultaneously (AVX2 `vpgatherdd`) is possible but gather latency is high — replicating the table across 16 registers and using `vpshufb` is often faster.

5. **Compression + ECC**: The external memory chapter covers the cost of data movement; error correction adds overhead bytes (redundancy) that consume additional memory bandwidth. A 10% redundancy rate means 10% fewer effective operations per byte of memory traffic — but the alternative (retransmission, corrupted data) is usually far more expensive.
