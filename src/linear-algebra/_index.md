# The Computational Backbone of Science

Linear algebra is not one algorithm — it's the substrate on which most of scientific computing runs. PDE solvers, optimization, statistics, machine learning, graphics, and quantum simulation all reduce to a handful of kernels: matrix multiplication, triangular solve, Cholesky/LU/QR factorization, and the SVD. Getting these kernels right (within 80% of hardware peak) is the difference between a simulation taking a day and taking a month.

The Matrix Multiplication case study (Chapter 11) already covered the core tiling and SIMD techniques. This chapter extends those ideas to the broader linear algebra landscape, with a focus on algorithms that exploit matrix structure (symmetry, sparsity, bandedness) and numerical stability considerations.

## What This Chapter Covers

1. **BLAS and LAPACK** — The three-level BLAS hierarchy. Why `dgemm` is 1000× faster than naive matrix multiply. The LAPACK factorization ecosystem. When to call a library vs. write your own.
2. **Triangular Systems and Cholesky** — Forward/back substitution. Cholesky factorization for SPD matrices. Blocked algorithms that achieve BLAS-3 speed. The 2× speedup from symmetry exploitation.
3. **LU Factorization and Gaussian Elimination** — Partial pivoting and numerical stability. Block LU with BLAS-3. The fundamental tradeoff: stability vs. performance.
4. **QR Factorization and Least Squares** — Householder reflections, Givens rotations, and Gram-Schmidt. The least-squares normal equations vs. QR. When to use the SVD instead.

## Recommended Reading Order

Read the **BLAS and LAPACK** article first — the three-level hierarchy explains the performance architecture of all subsequent algorithms. Then triangular systems/Cholesky (simplest factorizations), LU, and QR in order.

Cross-reference with Chapter 11 (Matrix Multiplication) for the underlying tiling+SIMD techniques, and Chapter 6 (Arithmetic) for floating-point error analysis.
