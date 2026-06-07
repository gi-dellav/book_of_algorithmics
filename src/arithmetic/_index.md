# Darker Corners of the ISA

x86-64 has somewhere between 1000 and 4000 instructions, depending on how you count. Most programmers use fewer than 50. The rest are "darker corners" — instructions and techniques that are specialized, obscure, or non-obvious, but can dramatically accelerate specific operations. This chapter explores them.

The arithmetic we do in code — adding numbers, comparing floats, dividing integers — seems simple. But the ISA offers multiple ways to do each, with performance characteristics that differ by factors of 10 or more. An integer division by a constant can be 10× faster when the compiler replaces it with multiplication. A bit-counting loop can be 50× faster with `popcnt`. A floating-point reduction can be 2× faster with FMA.

This chapter is not about mathematical correctness (that's assumed). It's about *how* the hardware does arithmetic, and how to choose the fastest way for your use case.

We cover:
- **Integer representations**: Signed vs. unsigned, two's complement, endianness, 128-bit integers.
- **IEEE 754**: The standard for floating-point arithmetic — formats, corner cases, encoding.
- **Floating-point in depth**: How to think about floats, from DIY float construction to hardware implementation.
- **Rounding errors**: Machine epsilon, Kahan summation, interval arithmetic, catastrophic cancellation.
- **Integer division**: Why division is slow, and how Barrett and Lemire reductions turn it into multiplication.
- **Newton's method**: Iterative root-finding with quadratic convergence.
- **Fast inverse square root**: The Quake III algorithm — a masterpiece of numerical approximation.
- **Bit hacks**: SWAR (SIMD Within A Register), popcount, bit reversal, and other bit-level techniques.
- **Fast math approximations**: Minimax polynomials, range reduction, `exp`/`log`/`sin`/`cos` approximations.
- **Data compression**: Run-length encoding, delta coding, varint encoding, SIMD-accelerated compression.

Each article ties back to the hardware: what the CPU actually does, how many cycles it takes, and what the alternatives cost.
