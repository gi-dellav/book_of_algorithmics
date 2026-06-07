# Cache-Oblivious Algorithms

A cache-oblivious algorithm has no parameters for cache size or cache line size. It doesn't tune itself to a specific machine. Yet it achieves asymptotically optimal I/O complexity across all levels of the memory hierarchy — automatically. This article covers the key algorithms and their analysis.

## The Principle

Divide and conquer. Recursively split the problem until the subproblems fit in cache — whatever size that cache happens to be. The recursion automatically adapts: on a machine with a large cache, the recursion bottoms out early; on a machine with a small cache, it goes deeper.

No explicit blocking. No B or M parameters in the code. The algorithm's I/O complexity is analyzed in the external memory model, and the bound holds for *all* values of B and M simultaneously. This is the magic of cache-obliviousness.

## Matrix Transposition

```c
void transpose_recursive(int *A, int *B, int n, int row_stride, int col_stride) {
    if (n <= 64) {  // Base case: fits in L1
        for (int i = 0; i < n; i++)
            for (int j = 0; j < n; j++)
                B[j * col_stride + i] = A[i * row_stride + j];
        return;
    }
    int half = n / 2;
    // Recursively transpose 4 quadrants
    transpose_recursive(A, B, half, row_stride, col_stride);
    transpose_recursive(A + half, B + half * col_stride, half, row_stride, col_stride);
    transpose_recursive(A + half * row_stride, B + half, half, row_stride, col_stride);
    transpose_recursive(A + half * row_stride + half, B + half * col_stride + half, half, row_stride, col_stride);
}
```

I/O analysis: Let I/O(n) be the I/O cost for n × n transpose. If n² ≤ M (the submatrix fits in cache), the base case costs at most ⌈n²/B⌉ I/Os (read once, write once). Otherwise, the recursion does 4 subproblems of size n/2, plus O(1) extra:

```
I/O(n) = 4 × I/O(n/2) + O(1)   if n² > M
I/O(n) = n²/B + O(1)           if n² ≤ M
```

Solution: I/O(n) = Θ(n²/B) when n² ≫ M — the same as the explicitly-blocked algorithm. The base case is when n² ≈ M/c for some constant c that depends on the recursion overhead (in practice, c ≈ 4 so the recursion stops when n ≈ √(M/4)).

## Cache-Oblivious Matrix Multiplication

```c
void matmul_recursive(float *C, float *A, float *B, int n, int stride) {
    if (n <= 32) {  // Base case
        for (int i = 0; i < n; i++)
            for (int k = 0; k < n; k++)
                for (int j = 0; j < n; j++)
                    C[i*stride + j] += A[i*stride + k] * B[k*stride + j];
        return;
    }
    int half = n / 2;
    // 8 recursive multiplications (standard Strassen-like decomposition)
    matmul_recursive(C, A, B, half, stride);              // C11 += A11 × B11
    matmul_recursive(C + half, A + half, B, half, stride); // C12 += A21 × B11 ??? (Actually more complex...)
    // ... (8 subproblems total for standard multiplication)
}
```

The standard recursive algorithm (not Strassen) does 8 subproblems of size n/2. Analysis shows the I/O complexity is Θ(N³/(B√M)) — which is optimal for matrix multiplication in the external memory model.

**Strassen's algorithm** in cache-oblivious form does 7 subproblems instead of 8 (adding and subtracting matrices). The I/O complexity is Θ(N^{log₂7}/(B × M^{log₂7/2−1})), asymptotically better than the 8-subproblem version.

The constant factors in the recursive version are large (function call overhead, pointer arithmetic). In practice, an explicitly-blocked version (like the one in `algorithms/matmul.md`) is 10–30% faster for a specific machine. The cache-oblivious version's advantage is portability — it runs well on any cache size without tuning.

## Cache-Oblivious B-Tree (Funnelsort / Cache-Oblivious Search Tree)

The cache-oblivious B-tree achieves O(log_{B+1} N) search cost and O((N/B) log_{M/B}(N/B)) sorting cost without knowing B or M. The structure is a recursively-defined "van Emde Boas layout":

- The tree is stored in a recursive array.
- The top subtree (size roughly √N) is stored contiguously.
- Each of the √N child subtrees is stored contiguously after the top subtree.
- Recursively within each subtree.

A search traverses the top subtree (which fits in cache for large enough N relative to M), then recurses into one child. The I/O cost is O(log_{B+1} N) — the same as an explicitly parameterized B-tree — because each level of the van Emde Boas recursion corresponds to roughly log_B levels of the search tree, and moving between recursion levels causes O(1) cache misses when the subtree fits in cache.

## Limitations

1. **Base case size**: If the base case is larger than M, the asymptotic bounds don't apply. The programmer must choose a base case that ensures the subproblem fits in cache. For L1 (32 KB), a base case of 64 × 64 integers (16 KB) is safe.

2. **Recursion overhead**: Function calls, stack management, and pointer arithmetic add constant-factor overhead. For very small subproblems, iteration is faster than recursion.

3. **Non-power-of-two sizes**: The recursion works cleanly for powers of two. For arbitrary sizes, padding or fractional recursion adds complexity.

4. **Not all problems are cache-oblivious**: Some problems (like FFT) resist cache-oblivious optimization. The best FFT implementations are explicitly tuned for specific cache sizes.

## When to Use

Cache-oblivious algorithms shine when:
- You're writing a library that must run on unknown hardware.
- The problem has a natural recursive structure (divide and conquer).
- The explicitly-tuned version would require many parameters.

They're less appropriate when:
- The problem is simple enough that a single blocking parameter suffices.
- You're optimizing for a known, fixed target.
- The recursion overhead dominates (e.g., for tiny matrices called in a hot loop).

The data structures chapter (`data-structures/`) applies cache-oblivious layouts to binary search, segment trees, and B-trees — demonstrating that the asymptotic benefits translate to real speedups.
