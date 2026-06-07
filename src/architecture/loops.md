# Loops and Conditionals

Loops are where programs spend most of their time. Understanding how they compile — and how small changes in your source code affect the generated assembly — is essential to writing fast code.

## Conditional Jumps and the FLAGS Register

Every conditional operation in x86-64 works through the FLAGS (or EFLAGS/RFLAGS) register. This is a 64-bit register where individual bits are set or cleared by arithmetic operations. The key flags:

| Bit | Name | Meaning |
|-----|------|---------|
| 0 | CF | Carry / unsigned overflow |
| 6 | ZF | Zero result |
| 7 | SF | Sign (copy of most significant bit) |
| 11 | OF | Signed overflow |

Arithmetic instructions (`add`, `sub`, `cmp`, `inc`, `dec`, `mul`, `imul`) set these flags. `cmp a, b` is exactly `sub a, b` but discarding the result — it computes `a - b` and sets flags accordingly. `test a, b` is `and a, b` without storing the result.

After the flags are set, conditional jump instructions decide whether to jump:

```asm
cmp rax, 0
je  label       ; jump if rax == 0 (ZF=1)
jne label       ; jump if rax != 0 (ZF=0)
jg  label       ; jump if rax > 0, signed (SF=OF and ZF=0)
jl  label       ; jump if rax < 0, signed (SF≠OF)
jge label       ; jump if rax >= 0, signed (SF=OF)
jle label       ; jump if rax <= 0, signed (SF≠OF or ZF=1)
```

## How a For Loop Compiles

```rust
for i in 0..n {
    sum += a[i];
}
```

The compiler generates one of two patterns.

**Pattern A: Count up, compare to n**

```asm
    xor ecx, ecx        ; i = 0
    test esi, esi       ; test n
    jle .Ldone          ; skip loop if n <= 0
.Lloop:
    add eax, [rdi + rcx*4]  ; sum += a[i]
    inc ecx                 ; i++
    cmp ecx, esi            ; compare i, n
    jl .Lloop               ; loop if i < n
.Ldone:
```

**Pattern B: Count toward zero** (preferred when n is known)

```asm
    mov ecx, esi        ; count = n
.Lloop:
    add eax, [rdi]      ; sum += *ptr
    add rdi, 4          ; ptr++
    dec ecx             ; count--
    jnz .Lloop          ; loop if count != 0
```

Pattern B is more efficient because:
- `dec` + `jnz` is one fewer instruction than `cmp` + `jl` (2 vs. 3 in the loop body).
- On some processors, `dec` + `jnz` can be macro-fused into a single µop.
- Index multiplication is eliminated (pointer increment instead of `[rdi + rcx*4]`).

When you write `for (int i = n-1; i >= 0; i--)`, you're hinting to the compiler that Pattern B is legal. Modern compilers can transform a count-up loop into count-down when they can prove the transformation is safe.

## Loop Unrolling

The compiler may unroll a loop — duplicate the body multiple times and adjust the counter accordingly — to reduce the overhead of the loop control instructions and expose more instruction-level parallelism.

```rust
for i in 0..n {
    sum += a[i];
}
```

After unrolling by 4:

```asm
    mov ecx, esi
    shr ecx, 2          ; count = n / 4
.Lloop:
    add eax, [rdi]
    add eax, [rdi + 4]
    add eax, [rdi + 8]
    add eax, [rdi + 12]
    add rdi, 16
    dec ecx
    jnz .Lloop
    ; ... handle remaining n % 4 elements ...
```

Four additions per decrement-branch pair means the branch overhead is 4× lower. But the loop body is larger, consuming more I-cache and decode bandwidth. The optimal unroll factor depends on the CPU's execution width and the cost of the loop body.

You can influence unrolling with pragmas:
```rust
// Unrolling hints: use #![feature(unroll_for_loops)] + #[unroll(4)] on nightly,
// or pass -C llvm-args=-unroll-count=4 to rustc.
for i in 0..n { /* ... */ }
```

Or with `-funroll-loops` (enabled at `-O3` in GCC). But compilers aren't always right — profile with and without unrolling for your specific case.

## The Cost of Conditional Branches

A conditional branch in the middle of a loop is expensive, but *why* depends on the branch predictor. If the same branch goes the same way 99% of the time, it costs almost nothing — the predictor learns the pattern and the pipeline stays full. If the branch is unpredictable (a fair coin flip each iteration), you pay the full misprediction penalty: roughly 15–20 cycles on Zen 2 while the pipeline is flushed and restarted.

This is the subject of `pipelining/branching.md`. For now, the key insight: a conditional branch that the CPU can predict is cheap. One it cannot predict costs ~20 cycles. If your loop body is 4 cycles long and you flip a coin each iteration, you're running at ~12 cycles per iteration (4 + 0.5 × 20). Remove the branch, and you're back at 4.

## Alternative Loop Patterns

**Duff's Device**: An extreme form of loop unrolling using `switch`-case fallthrough to handle the remainder. Rarely used today; compilers generate equivalent code.

**Sentinel loops**: Instead of checking `i < n` each iteration, place a sentinel value at the end of the array and loop until you hit it. Eliminates the comparison. Only works when the sentinel can't appear in normal data.

**Do-while loops**: `do { ... } while (condition);` guarantees at least one iteration, which lets the compiler place the loop condition at the bottom and avoid the initial `jmp` to check for zero iterations. Use when you know n > 0.

These micro-optimizations matter only in the hottest loops — and the compiler does most of them automatically. But knowing the patterns helps you understand what the compiler did, and debug cases where it didn't.
