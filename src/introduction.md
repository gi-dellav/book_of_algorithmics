# Introduction

When Donald Knuth began writing *The Art of Computer Programming* in 1962, he made a bet. The bet was that algorithms could be analyzed rigorously — that we could count operations, prove bounds, and predict performance from first principles. For three decades, the bet paid off. A faster algorithm meant fewer instructions, and fewer instructions meant less time.

Then the ground shifted.

Between 1990 and 2020, CPU clock speeds went from 33 MHz to roughly 5 GHz — a factor of 150. But the *ratio* of computation speed to memory speed went from perhaps 2:1 to well over 100:1. An instruction that touches memory can now be *hundreds of times* more expensive than one that stays in registers. The gap between a cache hit and a cache miss is larger than the entire execution time of most instructions. This means that Big O analysis — which treats all operations as equal — can be wrong by orders of magnitude.

The algorithm with worse asymptotic complexity can, and often does, run faster in practice. An O(n log n) algorithm that streams sequentially through memory will demolish an O(n) algorithm that chases pointers, for any n that fits in a real computer. This is not a small effect. It is *the* dominant effect in modern performance.

This book teaches you to understand that gap.

We start with the hardware. What actually happens inside a CPU when you run code? How does the pipeline work, and why does a mispredicted branch cost 15–20 cycles? What does a cache line look like, and why is a linear scan of 10,000 elements often faster than a binary search over the same data? We run experiments to measure these effects directly, because the numbers on the spec sheet rarely tell the full story.

From the hardware, we move up. We look at what compilers can and cannot do for you, and how to write code that they can optimize. We study profiling tools — not just how to run them, but how to interpret their output and avoid the common traps that produce misleading measurements. We explore the instruction set's darker corners: the fast inverse square root from Quake III, the Barrett reduction that turns division into multiplication, the bit hacks that let you count bits or reverse bytes with a handful of operations.

Then we apply all of this knowledge. We take concrete algorithms — matrix multiplication, integer factorization, binary search, segment trees, hash tables — and push them to their limits. A 50-line matrix multiplication routine that runs at 90% of BLAS performance. A binary search that is four times faster than `std::lower_bound` by eliminating branches and reordering memory. A factorization algorithm that cuts through 60-bit semiprimes in microseconds. Each case study is a worked example of the principles from the earlier chapters.

The book is organized to be read sequentially. Each chapter builds on the last, and cross-references connect related ideas across chapters. But you can also dive into individual case studies and follow the references back when you need to understand a technique.

A note on the code. Most examples are in C and C++, because these languages sit close to the hardware and expose the details we need. We use x86-64 assembly where necessary to show what the machine actually executes. The specific numbers — latencies, bandwidths, cache sizes — are measured on a Zen 2 processor (Ryzen 7 4700U) unless otherwise noted. If you have different hardware, the principles remain the same but the constants will differ. Run the benchmarks yourself. Change them. Break them. That's how you learn.

This is not a textbook. It is a practitioner's guide to making code run fast on real hardware. If you are a CS student who has taken a data structures course and written some C, you have everything you need. If you are a professional engineer who wants to understand why your carefully optimized code is still slower than it should be, this book is for you.

Let's start with the most important question: why is Big O not enough?
