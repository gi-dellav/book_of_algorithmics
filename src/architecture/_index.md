# Learning Architecture First

Here is a trap that catches most programmers who try to optimize code: they change things at random, measure the result, and hope for improvement. Sometimes it works. Usually it doesn't, because they don't understand what the machine is actually doing.

Learning computer architecture — even just the programmer's model of the CPU — is the single highest-leverage investment you can make before attempting optimization. When you understand how instructions are fetched, decoded, and executed, you can *predict* what will be fast before you write a line of code. When you understand the stack, calling conventions, and register allocation, you can look at a function signature and know whether inlining will help.

This chapter gives you that foundation. We start with the instruction set architecture — the contract between software and hardware. We then descend into assembly language, the human-readable representation of machine code. We cover loops and conditional branches, functions and the stack, indirect branching (virtual functions, switch statements), and machine code layout. Along the way, we look at both x86-64 and ARM64, since these two ISAs dominate modern computing.

## Why Assembly?

You don't need to *write* assembly. You need to *read* it. When the compiler produces code that's slower than expected, the explanation is always in the assembly output. Everything higher-level — your C++, your algorithmic choices, your data structure design — is translated into assembly before it runs. Understanding that translation is the difference between guessing and knowing.

A concrete example. Consider these two loops:

```c
// Version A
for (int i = 0; i < n; i++)
    if (a[i] > threshold)
        sum += a[i];

// Version B
for (int i = 0; i < n; i++)
    sum += (a[i] > threshold) ? a[i] : 0;
```

They compute the same result. Which is faster? If you understand branch prediction (covered in Chapter 3), you know the answer depends on whether `a[i] > threshold` is predictable. If you can read assembly, you can see that Version A compiles to a conditional jump and Version B compiles to a conditional move — and conditional moves don't depend on prediction at all.

That's the level of understanding this chapter aims for.

## What This Chapter Covers

1. **ISA** — RISC vs. CISC, the x86 and ARM families, and why the distinction still matters.
2. **Assembly** — Reading x86-64 and ARM64 assembly, registers, addressing modes, common idioms.
3. **Loops** — How loops compile, the FLAGS register, conditional jumps, and loop unrolling.
4. **Functions** — The stack, calling conventions, inlining, and tail call elimination.
5. **Indirect Branching** — Computed jumps, virtual function dispatch, switch statement optimization.
6. **Code Layout** — How machine code is arranged in memory, alignment, fetch/decode bandwidth, and `[[likely]]`/`[[unlikely]]`.
7. **Interrupts** — How the CPU communicates with the OS, system call mechanisms, and why syscalls are expensive.

Each article includes assembly snippets that you can verify yourself. Install a C compiler, pass `-S` to generate assembly, and follow along. Better yet, use [Compiler Explorer](https://godbolt.org) to see the assembly output live as you edit the code.

Let's begin with the instruction set itself.
