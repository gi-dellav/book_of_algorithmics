# Hardware History

To understand why modern computers work the way they do, you need to understand how we got here. The story of computer architecture over the last 60 years is a story of exponential growth colliding with physical limits — and the ingenuity that found ways around those limits.

## From Vacuum Tubes to Microchips

The first electronic computers — ENIAC (1945), UNIVAC I (1951) — used vacuum tubes. ENIAC had 17,468 tubes, consumed 150 kW of power, and occupied 1,800 square feet. It could perform about 5,000 additions per second. A modern smartphone performs approximately 10¹² operations per second while consuming 5 watts. The improvement factor is roughly 10¹¹ in performance per watt.

The transistor, invented at Bell Labs in 1947, changed everything. Transistors were smaller, faster, more reliable, and consumed dramatically less power than vacuum tubes. But the real revolution was the **integrated circuit**.

## Photolithography

Integrated circuits are made through photolithography. The process works something like this:

1. Start with a silicon wafer, polished to atomic flatness.
2. Coat the wafer with photoresist, a light-sensitive chemical.
3. Shine ultraviolet light through a mask — a stencil of the circuit pattern.
4. The exposed photoresist hardens (or softens, depending on the chemistry).
5. Wash away the unhardened resist, leaving the pattern.
6. Etch away the exposed silicon, or deposit new material onto it.
7. Repeat 50–100 times for different layers.

Each layer adds transistors, wires, or insulating material. The minimum feature size — the smallest line you can draw — determines how many transistors fit on a chip. In 1971, the Intel 4004 used a 10 μm process, fitting 2,300 transistors on a die. In 2020, TSMC's 5 nm process fits roughly 15 billion transistors on a similarly-sized die.

The feature size shrank by a factor of 2,000. The transistor count grew by a factor of 6.5 million.

## Moore's Law

In 1965, Gordon Moore — then at Fairchild Semiconductor, later co-founder of Intel — observed that the number of transistors on a chip had doubled roughly every year. He predicted this trend would continue for at least ten years. In 1975, he revised the estimate to doubling every two years.

Moore's law held for over 50 years. It was never a law of physics — it was a roadmap. The semiconductor industry organized itself around this cadence, coordinating research, investment, and manufacturing targets. Each generation of process technology enabled the next: smaller transistors meant faster switching, lower power consumption, and more transistors per chip.

## Dennard Scaling

Moore's law gave us more transistors. But what made them *useful* was Dennard scaling.

In 1974, Robert Dennard at IBM observed that as transistors shrink, their power density stays roughly constant. Cut a transistor's dimensions in half, and:

- The capacitance drops by half.
- The voltage can drop by half.
- The switching time drops by half (the transistor is faster).
- The power consumption drops by 4×.

This was a magical period. Each new process node gave you transistors that were smaller, faster, *and* more power-efficient — all at the same time. You could double the clock frequency, double the transistor count, and keep the same power budget. This is why clock speeds rose from 5 MHz to 5 GHz between 1980 and 2005.

## The Leakage Wall

Dennard scaling died around 2005. The reason was physics.

As transistors shrink below about 90 nm, quantum tunneling effects become significant. Electrons leak through the gate oxide even when the transistor is "off." This leakage current grows exponentially as the oxide gets thinner. At 65 nm and below, leakage power became comparable to switching power.

Voltage couldn't scale down further without the transistor failing to switch reliably. But if voltage doesn't scale down, power density *increases* with each shrink — more transistors switching at the same voltage in the same area produces more heat. Chips hit a thermal wall.

This is why clock speeds plateaued at roughly 4–5 GHz around 2005. Making a transistor switch faster requires more voltage, which produces more heat, which requires exotic cooling. The economic optimum settled at frequencies where air cooling suffices.

## Modern Approaches

With frequency scaling dead and Dennard scaling over, architects turned to new strategies:

**Pipelining**: Break instruction execution into stages (fetch, decode, execute, memory, write-back). Overlap the execution of multiple instructions, so the processor completes one instruction per cycle even though each instruction takes 5+ cycles. Modern CPUs have 14–19 pipeline stages.

**Superscalar Execution**: Execute multiple instructions in the same clock cycle. A modern x86 core can decode 4–6 instructions per cycle and dispatch them to 8–10 execution units.

**Out-of-Order Execution**: Don't execute instructions in program order. Execute them in whatever order their inputs become available, while maintaining the *illusion* of in-order execution. This requires a reorder buffer (ROB) that tracks up to ~200 in-flight instructions.

**SIMD**: Single Instruction, Multiple Data. One instruction operates on a vector of values. SSE (128-bit, 1999) handles 4 floats. AVX (256-bit, 2011) handles 8 floats. AVX-512 (512-bit, 2017) handles 16 floats.

**Multi-Core**: Since we couldn't make individual cores much faster, we put multiple cores on the same die. A modern server chip might have 64 cores, each running at 2–3 GHz.

**Caches**: Since memory couldn't keep up, we added layers of cache — L1 (32 KB, 4 cycles), L2 (512 KB, 12 cycles), L3 (8 MB, 40 cycles). These hide memory latency for well-behaved (sequential, predictable) access patterns.

**GPUs**: Graphics processors evolved into general-purpose massively parallel processors. A modern GPU has thousands of simple cores optimized for throughput rather than latency. Ideal for regular, data-parallel workloads.

**FPGAs and ASICs**: For extreme performance or efficiency, custom hardware can beat general-purpose processors by 10–100×. The cost is development time and flexibility.

## What This Means for Algorithms

Each of these innovations changes the cost model for algorithms:

- **Pipelining** punishes unpredictable branches.
- **Superscalar execution** rewards independent operations.
- **SIMD** rewards regular, dense data layouts.
- **Caches** punish random memory access and reward spatial/temporal locality.
- **Multi-core** rewards parallelism and punishes synchronization.

An algorithm designed for the RAM model ignores all of these effects. An algorithm designed for real hardware exploits them.

The rest of this book is about how.
