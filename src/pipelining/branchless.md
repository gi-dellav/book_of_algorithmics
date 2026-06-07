# Branchless Programming

If the branch predictor is the CPU's crystal ball, branchless programming is the art of not needing one. By replacing conditional branches with arithmetic and conditional moves, you eliminate branch mispredictions entirely. The tradeoff: you always compute both sides of the "branch," even when one side is unnecessary.

## The Arithmetic Trick

The simplest branchless technique uses the sign bit as a mask:

```c
// Branchy:
int result;
if (x < 50)
    result = x + 100;
else
    result = 0;
```

```c
// Branchless:
int mask = (x - 50) >> 31;  // mask = 0xFFFFFFFF if x < 50, 0x00000000 if x >= 50
int result = (x + 100) & mask;
```

How it works: `x - 50` is negative if `x < 50` (MSB = 1). Shifting right 31 bits (arithmetic shift, `sar` in x86) fills all 32 bits with the sign bit, creating a mask that's all-ones (which is -1 in two's complement) or all-zeros (0). AND-ing with the mask gives us the value when the condition is true, and 0 when it's false.

This is fragile. It only works with 32-bit signed integers. It breaks if `x - 50` overflows. And the compiler may already generate exactly this code for a ternary expression that is amenable to `cmov`.

## Conditional Move (`cmov`)

x86-64 has a family of conditional move instructions:

```asm
cmp eax, 50
cmovge ebx, ecx   ; ebx = ecx if eax >= 50 (signed), else unchanged
```

`cmov` reads both operands, evaluates a condition (from FLAGS), and writes the destination register only if the condition is true. It looks like a branch but isn't — the CPU executes the `cmov` as a single dataflow operation, with no control flow change. The pipeline stays full.

The compiler generates `cmov` for:
```c
result = (condition) ? true_value : false_value;
```
when both `true_value` and `false_value` are simple (no side effects, no memory access, no function calls).

```c
// This generates cmov:
x = (a > b) ? a : b;  // cmovg after cmp

// This generates a branch:
x = (a > b) ? expensive_function() : 0;  // compiler won't compute expensive_function() unconditionally
```

You can check the assembly to see which path the compiler took. If you find a branch where you expected `cmov`, the compiler judged the "unconditional" path too expensive.

## When Branchless Wins

The threshold depends on the branch misprediction rate and the cost of computing both sides.

Let:
- `C_branch_correct` = 1 cycle (branch correctly predicted, both sides cheap)
- `C_branch_mispredict` = 20 cycles
- `C_branchless` = cost to compute both sides + `cmov` (~3–4 cycles if both sides are simple)
- `p` = probability branch is taken

Expected cost:
- Branch: 1 + (1 − max(p, 1−p)) × 20
- Branchless: 4

Break-even: `1 + (1 − max(p, 1−p)) × 20 = 4` → `(1 − max(p, 1−p)) = 0.15` → `max(p, 1−p) = 0.85`

**If the branch is predictable more than 85% of the time, branching wins. Otherwise, branchless wins.** (These numbers are for Zen 2; adjust for your microarchitecture.)

In practice:
- Sorted data, predictable condition → branch.
- Random data, unpredictable condition → branchless.
- Data dependent condition where you don't know predictability → measure both.

## Real-World Branchless Patterns

### Branchless Min/Max

```c
// Branchy
int min = (a < b) ? a : b;

// Compiler generates cmov (usually). But check assembly.
```

### Branchless Absolute Value

```c
// Without branch (two's complement trick):
int mask = x >> 31;        // all-ones if negative
int abs = (x ^ mask) - mask;  // ~x + 1 if negative, x if positive
// Or: (x + mask) ^ mask

// Or trust the compiler:
int abs = (x < 0) ? -x : x;  // often generates cmov, not a branch
```

### Branchless Binary Search

The standard binary search has a branch inside the loop:
```c
while (lo < hi) {
    int mid = (lo + hi) / 2;
    if (a[mid] < target)       // branch!
        lo = mid + 1;
    else
        hi = mid;
}
```

The branchless version uses `cmov`:
```c
while (lo < hi) {
    int mid = (lo + hi) / 2;
    // Always compute both possibilities, select with cmov
    int new_lo = mid + 1;
    int new_hi = mid;
    lo = (a[mid] < target) ? new_lo : lo;
    hi = (a[mid] < target) ? hi : new_hi;
}
```

For small arrays (n < 256, fitting in L1/L2 cache), the branchless version is ~2–4× faster because the data is unpredictable (search keys are arbitrary). For large arrays where cache misses dominate, the branch predictor is not the bottleneck, and the standard version is fine.

Chapter 12 (`data-structures/binary-search.md`) explores this optimization in depth, including the Eytzinger memory layout that makes branchless binary search even faster.

### Data-Parallel Masking (SIMD)

SIMD instructions naturally support branchless operations via masking:

```c
// Instead of:
for (int i = 0; i < n; i++)
    if (a[i] > 0)
        b[i] = sqrt(a[i]);
    else
        b[i] = 0;

// SIMD with masking:
__m256 zero = _mm256_setzero_ps();
__m256 mask = _mm256_cmp_ps(a_vec, zero, _CMP_GT_OQ);  // all-ones where > 0
__m256 result = _mm256_sqrt_ps(a_vec);                   // compute sqrt for all
result = _mm256_and_ps(result, mask);                    // zero where condition fails
```

Four operations, no branches, 8 elements at a time. The `sqrt` is computed for all elements (including those we'll mask out), but the throughput gain from SIMD outweighs the wasted work.

## When NOT to Go Branchless

1. **Expensive computation on the rare path**: If the "true" case takes 100 cycles and happens 1% of the time, computing it unconditionally is insane.

2. **Memory access on both paths**: If both sides of `cmov` read from memory, both memory accesses happen unconditionally. This is worse than a correctly-predicted branch that skips one.

3. **Side effects**: If either path has side effects (writes to memory, I/O), you can't use `cmov` — the compiler won't generate it, and you shouldn't force it.

4. **When the branch is perfectly predictable**: The loop counter (`i < n`) is predicted with ~100% accuracy. Don't branchless-ify your loop counter.

## The Compiler Is Your Friend

Modern compilers know about branchless techniques and apply them when profitable. Before writing manual bit-twiddling, check what the compiler generates for a simple ternary:

```bash
gcc -O2 -S your_code.c -o - | less
```

If you see `cmov`, the compiler already did the work. If you see `jcc` (conditional jump), the compiler judged the branch more efficient — and it's probably right for the general case. Only override it when profiling shows a specific, predictable misprediction problem.
