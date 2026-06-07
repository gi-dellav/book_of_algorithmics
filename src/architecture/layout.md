# Machine Code Layout

How machine code is arranged in memory affects performance. The CPU's front-end fetches and decodes instructions in chunks. If hot code spans multiple cache lines, is misaligned, or is interleaved with cold code, the front-end wastes bandwidth and the instruction cache fills with dead bytes. This article covers code alignment, fetch/decode bandwidth, unequal branches, `cmov`, and the `[[likely]]`/`[[unlikely]]` attributes.

## The CPU Front-End

Before an instruction executes, it must be:

1. **Fetched** from the instruction cache (L1i, typically 32 KB, 8-way set-associative).
2. **Decoded** into µops (micro-operations) by the instruction decoder.
3. **Stored** in the µop cache (if available) for reuse.

On Zen 2, the front-end fetches 32 bytes per cycle from the L1i cache. The decoders produce up to 4 µops per cycle. There is also a µop cache that stores up to 4096 previously-decoded µops — if your hot loop fits entirely in the µop cache, the fetch and decode stages are skipped entirely.

If the front-end can't keep up, the back-end (execution units) starve for work, no matter how fast they are. Code layout affects front-end throughput.

## Code Alignment

Instructions are aligned by the compiler to improve fetch efficiency. The x86-64 ISA allows instructions to start at any byte, but the fetch engine works best when branch targets and function entries are aligned to 16 or 32 bytes.

```asm
    nop                     ; padding
    nop
    .align 16
.Lloop_start:               ; aligned to 16 bytes
    add eax, [rdi]
    add rdi, 4
    dec ecx
    jnz .Lloop_start
```

Why does alignment matter? If a hot loop starts near the end of a 32-byte fetch window, the CPU might need two fetch cycles to get the first few instructions — effectively cutting fetch bandwidth in half each iteration.

The compiler inserts `nop` instructions to align branch targets. These no-ops cost nothing when not executed (they're skipped by the branch) but consume code space. The optimal alignment is a tradeoff: 16-byte alignment is good for most loops; 32-byte for extremely hot loops; avoid alignment for cold code.

Recent CPUs (Zen 4, Intel Golden Cove) have more sophisticated fetch units that make alignment less critical than it was a decade ago. But the principle still holds: don't fight the front-end.

## Fetch and Decode Bandwidth

The instruction cache line is 64 bytes. The Zen 2 fetch engine reads 32 bytes per cycle. For a loop to execute at one iteration per cycle, the average instruction length must be ≤ 32 bytes / IPC, where IPC is instructions per cycle.

x86-64 instructions average 3–4 bytes. At 32 bytes/cycle, the front-end can sustain ~8 instructions per cycle — more than the decoder's 4 µop/cycle limit. So the decoder, not the fetch engine, is usually the bottleneck for scalar code.

But SIMD code can stress the decode limit. AVX instructions are longer (often 5–8 bytes with VEX/EVEX prefixes) and each instruction does more work. A loop that processes 32 bytes of data with 3 instructions per iteration needs 15–24 bytes of instruction encoding and takes 1 fetch cycle — fine. If the loop body is larger (many shuffles, extract/insert operations), the front-end becomes the bottleneck.

## Unequal Branches and `cmov`

```c
if (condition)
    x = expensive_computation();
else
    x = 0;
```

If the condition is predictable, this is fine. If it's unpredictable, the branch mispredicts half the time, paying ~20 cycles each time.

The compiler may transform this into a conditional move (`cmov`):

```asm
    xor eax, eax        ; eax = 0 (else case)
    cmp edi, 0          ; test condition
    cmovne eax, ebx     ; if condition true, eax = expensive_result
```

`cmov` evaluates *both* branches' values and selects between them without branching. It avoids the misprediction penalty — but it always computes both sides. If `expensive_computation()` is truly expensive (division, memory access), computing it unconditionally may be worse than the occasional mispredict.

The compiler's heuristic: use `cmov` when both sides are cheap (a few cycles). Otherwise, use a branch. You can override this with `[[likely]]` / `[[unlikely]]` to tell the compiler which path is hot, or use the ternary operator `condition ? true_val : false_val` (which the compiler may compile to `cmov`).

## `[[likely]]` and `[[unlikely]]`

C++20 attributes that hint the branch direction to the compiler:

```cpp
if (__builtin_expect(x == 0, 0))  // C-style (GCC)
    handle_rare_case();

if (x == 0) [[unlikely]]          // C++20
    handle_rare_case();
```

These attributes affect two things:
1. **Code layout**: The `[[likely]]` path is placed sequentially with the surrounding code; the `[[unlikely]]` path is placed elsewhere (often after the function's return). This improves I-cache utilization — the hot path is contiguous.
2. **Branch prediction hints**: On older x86 processors, a prefix byte could hint the branch predictor. Modern processors ignore these hints; they use dynamic prediction instead. The layout effect is the primary benefit.

```asm
; Without [[unlikely]]:
    test rax, rax
    jz .Lrare_case     ; branch for rare case — hot path falls through
.Lcommon:
    ...

; With [[unlikely]]:
    test rax, rax
    jnz .Lcommon        ; branch for common case — rare case falls through
    call handle_rare_case
.Lcommon:
    ...
```

The second form puts the rare case out-of-line, keeping the common case contiguous.

## The Perils of Out-of-Line Code

Large functions (especially with many `[[unlikely]]` branches) can scatter code across I-cache lines, causing fetch stalls even on the hot path. When you see a function with many uncommon paths, consider:

- Moving cold code to separate functions (with `__attribute__((noinline))` on the cold side).
- Using `__builtin_expect` to train the compiler's layout heuristics.
- Checking the binary with `perf record` / `perf report` to see if I-cache misses are significant.

The golden rule: **the hot path should be contiguous in memory.** Every jump out and back is an opportunity for the front-end to lose its place.

## Interaction with the Pipelining Chapter

This article previews concepts (branch prediction, pipeline flushes) that are covered in detail in Chapter 3 (`pipelining/`). Code layout is the spatial aspect of branch handling; branch prediction is the temporal aspect. Both matter for performance, and they interact: a correctly-predicted branch that jumps to a cold cache line still costs a fetch stall, even though the pipeline doesn't flush.
