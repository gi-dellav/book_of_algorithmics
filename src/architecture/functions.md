# Functions and the Stack

Functions are the fundamental unit of code organization. They also introduce runtime overhead: calling conventions, stack frame setup/teardown, register save/restore, and the lost optimization opportunity when the compiler can't see through a call boundary. Understanding this machinery is essential for deciding when to inline, when to outline, and how to structure hot code.

## The Stack

The stack is a region of memory that grows downward (toward lower addresses). `rsp` points to the top of the stack (the lowest occupied address). `push` decrements `rsp` and writes to the new top; `pop` reads from the top and increments `rsp`.

```
High addresses
+-------------------+
|     ...           |
|   argument N      |  <- [rsp + 8*N + 8]
|   argument 1      |  <- [rsp + 16]
|   return address  |  <- [rsp + 8]
|   saved rbp       |  <- [rsp]  (frame pointer, optional)
|   local variables |
|   ...             |
+-------------------+
Low addresses       <- rsp (stack pointer)
```

Each function call allocates a **stack frame** — a contiguous region on the stack that holds the function's local variables, saved registers, and (for non-leaf functions) the return address.

## Calling Conventions

The calling convention specifies how arguments are passed, how return values are returned, and which registers the callee must preserve. On x86-64 Linux/macOS (System V AMD64 ABI):

**Integer/pointer arguments** (in order): `rdi`, `rsi`, `rdx`, `rcx`, `r8`, `r9`. Additional arguments go on the stack.

**Floating-point arguments** (in order): `xmm0`–`xmm7`.

**Return value**: Integer/pointer in `rax`. Floating-point in `xmm0`.

**Callee-saved registers**: `rbx`, `rbp`, `r12`–`r15`. The callee must restore these before returning. The caller can assume they survive a call.

**Caller-saved registers**: Everything else (`rax`, `rcx`, `rdx`, `rsi`, `rdi`, `r8`–`r11`, `xmm0`–`xmm15`). The caller must save these if their values are needed after the call.

**Shadow space** (Windows only): 32 bytes reserved by the caller for the callee to spill register arguments. Not present on Linux/macOS.

**Stack alignment**: `rsp` must be 16-byte aligned at the `call` instruction (i.e., at function entry, `rsp` is 8 mod 16 because `call` pushed an 8-byte return address).

ARM64 (AArch64 ABI) is similar but with different register mappings:
- Arguments: `x0`–`x7` (integer), `v0`–`v7` (floating-point).
- Return value: `x0`/`v0`.
- Callee-saved: `x19`–`x30` (includes frame pointer `x29` and link register `x30`).

## Function Prologue and Epilogue

```c
long foo(long a, long b) {
    long c = a + b;
    return c * c;
}
```

Compiles to (x86-64, `-O2`):

```asm
foo:
    lea rax, [rdi + rsi]    ; rax = a + b
    imul rax, rax           ; rax *= rax
    ret
```

No stack frame needed. The function is a **leaf** (calls no other functions) and uses no callee-saved registers, so it can operate entirely in argument registers and caller-saved registers.

Now a function that needs the stack:

```c
long bar(long a, long b) {
    long c = foo(a, b);     // call to another function
    return c + a + b;
}
```

```asm
bar:
    push rbx                ; save callee-saved rbx
    mov rbx, rdi            ; rbx = a (preserve across call)
    push rsi                ; save b (not callee-saved, but needed after call)
    call foo                ; rax = foo(a, b)
    add rax, rbx            ; rax += a
    pop rsi                 ; restore b into rsi (not needed, just balancing stack)
    pop rbx                 ; restore rbx
    add rax, rsi            ; rax += b
    ret
```

Key points:
- `a` is in `rdi` when we enter, but `rdi` is caller-saved — `foo` might clobber it. So we save `a` into `rbx` (callee-saved) before the call.
- `b` is in `rsi`, also caller-saved, so we `push` it and `pop` it after.
- Actually the compiler could do better: just save both registers with `push rdi; push rsi` and avoid the `rbx` move. In practice the compiler optimizes this further.

When frame pointers are used (`-fno-omit-frame-pointer`, default in debug builds):

```asm
bar:
    push rbp
    mov rbp, rsp
    sub rsp, 16        ; allocate 16 bytes of local storage
    ...
    leave              ; mov rsp, rbp; pop rbp
    ret
```

`rbp` chains stack frames together, enabling debuggers and profilers to unwind the stack without DWARF metadata. With `-O2`, the compiler omits the frame pointer and uses DWARF/`.eh_frame` metadata instead, saving two instructions per function.

## Inlining

Inlining replaces a function call with the body of the callee. It eliminates call/ret overhead, enables cross-function optimizations (constant propagation, dead code elimination), and reduces register pressure from calling conventions.

```c
static inline long foo_inline(long a, long b) {
    return (a + b) * (a + b);
}

long baz(long a, long b) {
    return foo_inline(a, b);
}
```

After inlining, `baz` becomes:

```asm
baz:
    lea rax, [rdi + rsi]
    imul rax, rax
    ret
```

The `call`/`ret` overhead disappears. More importantly, if `baz` is called with a constant argument, the compiler can fold it: `baz(3, 5)` → `foo_inline(3, 5)` → `(3+5)*(3+5)` → `64` at compile time.

The compiler decides whether to inline based on heuristics: function size, call frequency, and optimization level. You can influence it:
- `static inline` strongly suggests inlining (but doesn't guarantee it).
- `__attribute__((always_inline))` forces inlining.
- `__attribute__((noinline))` prevents it (useful for profiling).
- `-finline-limit=N` adjusts the size threshold.

Over-inlining bloats code size, increasing I-cache pressure. Profile-guided optimization (PGO) gives the compiler accurate call counts and helps it make better inlining decisions.

## Tail Call Elimination

When a function's last action is calling another function, the compiler can replace `call` + `ret` with a single `jmp`:

```c
long tail_add(long n, long acc) {
    if (n == 0) return acc;
    return tail_add(n - 1, acc + 1);
}
```

With `-O2`:

```asm
tail_add:
    test rdi, rdi
    je .Ldone
.Lloop:
    add rsi, 1
    dec rdi
    jnz .Lloop
.Ldone:
    mov rax, rsi
    ret
```

The compiler transformed the recursive call into a loop. The recursive structure became iteration with no stack growth. Tail call elimination makes certain recursive algorithms (accumulator-passing style) as efficient as loops.

Key requirement: the call must be in **tail position** — the caller must not need to do any work after the callee returns. The return value of the callee must be the return value of the caller, unmodified.
