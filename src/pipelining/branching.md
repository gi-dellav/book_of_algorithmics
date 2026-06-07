# Branch Prediction

Modern CPUs execute billions of instructions per second. To keep the pipeline full, they must guess the outcome of every conditional branch *before* the condition is evaluated. They are astonishingly good at this — prediction accuracy exceeds 95% for most code — but when they're wrong, the cost is severe.

## The Experiment

This is the experiment that first made branch prediction visceral for a generation of programmers. It's been passed around Stack Overflow, blog posts, and conference talks. We'll run it ourselves.

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

int main() {
    const int n = 32768;
    int *a = malloc(n * sizeof(int));
    
    // Fill array with random numbers
    for (int i = 0; i < n; i++)
        a[i] = rand() % 256;
    
    // Sum elements greater than 128
    long long sum = 0;
    clock_t start = clock();
    for (int i = 0; i < 100000; i++) {
        for (int j = 0; j < n; j++) {
            if (a[j] > 128)
                sum += a[j];
        }
    }
    clock_t end = clock();
    printf("Unsorted: %f seconds\n", (double)(end - start) / CLOCKS_PER_SEC);
    
    // Sort the array
    qsort(a, n, sizeof(int), (int(*)(const void*, const void*))cmp);
    
    // Repeat with sorted array
    sum = 0;
    start = clock();
    for (int i = 0; i < 100000; i++) {
        for (int j = 0; j < n; j++) {
            if (a[j] > 128)
                sum += a[j];
        }
    }
    end = clock();
    printf("Sorted: %f seconds\n", (double)(end - start) / CLOCKS_PER_SEC);
}
```

On Zen 2, the sorted version runs roughly **4–6× faster** than the unsorted version. The only difference is the order of elements. The condition `a[j] > 128` is true for approximately half the elements in both cases. But in the sorted array, all the "less than 128" elements are grouped together, and all the "greater than 128" elements are grouped together. The branch predictor detects this pattern and achieves near-100% accuracy. In the random array, the branch is a coin flip — the predictor is wrong 50% of the time.

## How Branch Prediction Works

Modern branch predictors combine multiple techniques:

**Branch History Table (BHT)**: A cache of recently-seen branches, indexed by low bits of the branch instruction's address. Each entry stores the outcome of the last N executions (typically 1–2 bits for a saturating counter: strongly taken, weakly taken, weakly not-taken, strongly not-taken). Simple and effective for loops with consistent behavior.

**Global History**: A shift register of the outcomes of the last K branches. Combined with the branch address, this detects patterns like "branch A taken, branch B taken, branch A not-taken" — correlated branches that a per-branch history would miss.

**Loop Counter**: A dedicated predictor for loops that execute a fixed number of iterations. Detects the "taken, taken, ..., taken, not-taken" pattern and predicts the loop exit accurately every time.

**Perceptron / Neural Predictors**: Modern high-end CPUs (Zen 2+, Intel since Haswell) use perceptron-based predictors that can learn long, complex patterns. They treat branch prediction as a classification problem: given the history (random linear combination of past branch outcomes and addresses), predict taken or not-taken. This handles patterns that saturating counters miss.

**Indirect Branch Predictor**: For `jmp rax` and `call [rax]`, a separate structure predicts the target address, not just the direction. It learns that "this virtual call usually goes to `DerivedFoo::process`."

## Prediction Quality vs. Probability

The cost of a misprediction is roughly 15–20 cycles (Zen 2). The cost of a correct prediction is ~1 cycle (the branch itself executes, but execution overlaps with fetch).

If a branch has probability p of being taken:

- **Expected cost per branch**: 1 + (1 − prediction_accuracy) × 20 cycles.

Where prediction accuracy = max(p, 1−p) for a simple predictor. For a sophisticated predictor, accuracy can exceed max(p, 1−p) if there's a detectable pattern.

- p = 0.50 (fair coin): accuracy ≈ 0.50 → cost ≈ 1 + 0.50 × 20 ≈ 11 cycles.
- p = 0.90: accuracy ≈ 0.90 → cost ≈ 1 + 0.10 × 20 ≈ 3 cycles.
- p = 0.99: accuracy ≈ 0.99 → cost ≈ 1 + 0.01 × 20 ≈ 1.2 cycles.

**The 75% threshold**: At p ≈ 0.75, a predicted branch costs about the same as a conditional move (~4 cycles on Zen 2). Below 75% predictability, `cmov` wins. Above 75%, the branch wins. This threshold varies by microarchitecture but the principle holds: unpredictable branches should be eliminated.

## Making Branches Predictable

1. **Sort data before processing**: If you're branching on a property of elements, sorting groups identical outcomes together, creating long runs that predictors easily learn.

2. **Process hot and cold data separately**: Split your data into "needs special handling" and "fast path" subsets. Process each with a dedicated loop — no branches inside the loop.

3. **Use `[[likely]]` / `[[unlikely]]`**: These don't change prediction behavior on modern CPUs (the predictor is dynamic), but they improve code layout — the hot path stays contiguous, reducing front-end stalls.

4. **Separate error handling**: Put error checks outside the hot loop. If an error might occur inside, move the check outside and process the error cases separately.

5. **Use sentinel values**: Instead of checking `i < n` every iteration (a branch that mispredicts exactly once per loop), place a sentinel value at the end of the array and loop until you hit it.

## The Limits of Prediction

Branch predictors have finite state. Zen 2's branch predictor can track approximately 1024 branches simultaneously. If your code has more active branches than that, older entries are evicted and re-learned — causing mispredicts on what should be perfectly predictable branches.

The indirect branch predictor has even less state. If you make virtual function calls to more than ~16–32 different targets through the same call site, prediction accuracy degrades.

When prediction fails, the solution is not to get a better predictor — it's to *remove the branch*. The next article covers branchless programming.
