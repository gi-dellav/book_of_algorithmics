# Assembly Language

Assembly is the human-readable representation of machine code. This article teaches you to read x86-64 assembly well enough to debug performance problems. We'll cover ARM64 when the differences matter.

## Syntax Wars: AT&T vs. Intel

x86 assembly comes in two syntaxes. You need to recognize both.

**AT&T syntax** (GCC/GAS default):
```asm
movl $5, -4(%rbp)       # move 5 to [rbp-4]
addl %eax, %ebx         # ebx += eax
```
- Source first, destination second.
- Register names prefixed with `%`.
- Immediate values prefixed with `$`.
- Operand size encoded as instruction suffix (`l` = long/32-bit, `q` = quad/64-bit, `w` = word/16-bit, `b` = byte).

**Intel syntax** (used by most documentation, Compiler Explorer default):
```asm
mov dword [rbp-4], 5    ; move 5 to [rbp-4]
add ebx, eax            ; ebx += eax
```
- Destination first, source second.
- Registers without prefix.
- Immediates without prefix.
- Size specified on memory operand (`dword`, `qword`, `word`, `byte`).

This book uses Intel syntax. If you see `%` and `$`, you're looking at AT&T syntax. Swap the operands mentally.

## Registers

The x86-64 register file:

| 64-bit | 32-bit | 16-bit | 8-bit | Purpose |
|--------|--------|--------|-------|---------|
| `rax` | `eax` | `ax` | `al` | Return value, accumulator |
| `rbx` | `ebx` | `bx` | `bl` | Callee-saved |
| `rcx` | `ecx` | `cx` | `cl` | 4th argument, counter |
| `rdx` | `edx` | `dx` | `dl` | 3rd argument, division high bits |
| `rsi` | `esi` | `si` | `sil` | 2nd argument, source index |
| `rdi` | `edi` | `di` | `dil` | 1st argument, dest index |
| `rbp` | `ebp` | `bp` | `bpl` | Frame pointer (callee-saved) |
| `rsp` | `esp` | `sp` | `spl` | Stack pointer |
| `r8`–`r15` | `r8d`–`r15d` | `r8w`–`r15w` | `r8b`–`r15b` | General purpose |

Writing to a 32-bit register zero-extends to 64 bits. Writing `eax` sets the high 32 bits of `rax` to zero. This is *not* true for 8/16-bit writes — `mov al, 5` leaves the upper 56 bits of `rax` unchanged. The zero-extension rule enables compact zeroing: `xor eax, eax` zeros `rax` in 2 bytes.

## Common Instructions

### Data Movement

```asm
mov rax, rbx        ; rax = rbx
mov rax, [rbx]      ; rax = *rbx (load 8 bytes from memory at address rbx)
mov [rbx], rax      ; *rbx = rax (store 8 bytes to memory at address rbx)
mov rax, 42         ; rax = 42 (immediate)
lea rax, [rbx+rcx*8] ; rax = rbx + rcx*8 (no memory access — just arithmetic)
```

`lea` (Load Effective Address) is an arithmetic instruction that happens to use the addressing-mode syntax. It computes an address but doesn't access memory. `lea rax, [rbx + rcx*8 + 16]` is a compact way to do `rax = rbx + rcx*8 + 16` in one instruction.

### Arithmetic

```asm
add rax, rbx        ; rax += rbx
sub rax, rbx        ; rax -= rbx
imul rax, rbx       ; rax *= rbx (signed, truncated to 64 bits)
xor rax, rax        ; rax = 0 (idiomatic zeroing, smaller encoding than mov)
inc rax             ; rax++
shl rax, 3          ; rax <<= 3
sar rax, 1          ; rax >>= 1 (arithmetic, sign-extending)
shr rax, 1          ; rax >>= 1 (logical, zero-extending)
```

Integer division is special: `idiv rbx` divides the 128-bit value `rdx:rax` by `rbx`, placing the quotient in `rax` and remainder in `rdx`. Before unsigned division, zero `rdx` with `xor edx, edx`. Before signed division, sign-extend `rax` into `rdx` with `cqo` (Convert Quadword to Octoword). Division is slow — 13–44 cycles on Zen 2 — so compilers avoid it when possible (see the arithmetic chapter).

### Comparison and Conditionals

```asm
cmp rax, rbx        ; compute rax - rbx, set FLAGS, discard result
test rax, rax       ; compute rax & rax, set FLAGS, discard result (test for zero)
je label            ; jump to label if ZF=1 (equal / zero)
jne label           ; jump if ZF=0 (not equal)
jg label            ; jump if greater (signed)
jl label            ; jump if less (signed)
ja label            ; jump if above (unsigned)
jb label            ; jump if below (unsigned)
```

The FLAGS register stores the outcome of arithmetic: ZF (zero), SF (sign/negative), CF (carry/unsigned overflow), OF (signed overflow). `cmp` subtracts without storing the result, setting FLAGS. Conditional jumps read FLAGS.

### Function Calls

```asm
call func           ; push rip, jump to func
ret                 ; pop rip, jump to that address
push rax            ; rsp -= 8, [rsp] = rax
pop rax             ; rax = [rsp], rsp += 8
```

## Addressing Modes

x86-64 supports this general form: `[base + index*scale + displacement]`

- `base`: any register
- `index`: any register except `rsp`
- `scale`: 1, 2, 4, or 8
- `displacement`: signed 8-bit or 32-bit constant

Examples:
```asm
mov rax, [rbx]              ; base only
mov rax, [rbx + 16]         ; base + displacement
mov rax, [rbx + rcx*8]      ; base + scaled index
mov rax, [rbx + rcx*8 + 16] ; all three
mov rax, [rip + offset]     ; RIP-relative (used for global variables)
```

## Practical: Reading Compiler Output

Consider this C function:

```rust
unsafe fn sum(arr: *const i32, n: i32) -> i32 {
    let mut total = 0;
    for i in 0..n {
        total += *arr.add(i as usize);
    }
    total
}
```

Compiled with `gcc -O2 -march=znver2`, Intel syntax:

```asm
sum:
    test esi, esi          ; test n, n
    jle .L4                ; if n <= 0, return 0
    lea eax, [rsi-1]       ; eax = n - 1
    lea rdx, [rdi+rax*4+4] ; rdx = &arr[n] (past-the-end)
    xor eax, eax           ; total = 0
.L3:
    add eax, [rdi]         ; total += *arr
    add rdi, 4             ; arr++
    cmp rdi, rdx           ; while arr != past-the-end
    jne .L3
    ret
.L4:
    xor eax, eax           ; return 0
    ret
```

Observations:
- The compiler transformed `arr[i]` into a pointer increment (`add rdi, 4`), eliminating the index multiplication.
- It computes the past-the-end address once (`lea rdx, [rdi+rax*4+4]`) rather than comparing `i < n` each iteration.
- `test esi, esi; jle .L4` handles the n ≤ 0 case upfront.
- With AVX2 (`-mavx2`), this loop would use `vpaddd` to process 8 integers at once.

The habit of checking the compiler's output — even when you trust it — will save you from assumptions about what your code actually does.
