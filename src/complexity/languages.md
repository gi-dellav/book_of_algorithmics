# Languages and Performance

Here is the single most important demonstration in this book. We will run matrix multiplication — the same algorithm, the same operation count — in different languages and configurations, on the same hardware. The performance difference between the slowest and fastest version is a factor of 5,250.

## The Algorithm

We multiply two n × n matrices A and B to produce C = A × B. The inner loop does this:

```rust
for i in 0..n {
    for k in 0..n {
        for j in 0..n {
            c[i][j] += a[i][k] * b[k][j];
        }
    }
}
```

The operation count is exactly 2n³ floating-point operations (n³ multiplies + n³ adds). All versions do the same math. The only difference is how the code is compiled, optimized, and executed.

We test with n = 1024 on a Zen 2 core at 2.0 GHz. The theoretical peak for one core is 32 GFLOPS (16 single-precision FMA operations per cycle at 2 GHz, since Zen 2 has two 256-bit FMA units).

## Python (Pure)

```python
for i in range(n):
    for j in range(n):
        for k in range(n):
            C[i][j] += A[i][k] * B[k][j]
```

**Time: ~630 seconds. Performance: ~3.4 MFLOPS.**

Python is an interpreted language. Each line of the inner loop involves dictionary lookups, dynamic type checks, integer object allocation, and reference counting. The overhead of the interpreter is roughly 10,000× the cost of the actual floating-point operation. At 3.4 MFLOPS, the program achieves 0.01% of the CPU's theoretical peak.

Nobody should write matrix multiplication in pure Python. But this number establishes the baseline: the *same algorithm* can vary by four orders of magnitude depending on how it's executed.

## Java (HotSpot JIT)

```java
for (int i = 0; i < n; i++)
    for (int j = 0; j < n; j++)
        for (int k = 0; k < n; k++)
            c[i][j] += a[i][k] * b[k][j];
```

**Time: ~10 seconds. Performance: ~210 MFLOPS.**

The Java Virtual Machine's just-in-time compiler transforms the bytecode into native machine code at runtime. The inner loop becomes tight x86-64 assembly — loads, multiplies, adds, stores. There is no interpreter overhead. But Java's array bounds checks add a few instructions per iteration, and the JIT compiler cannot vectorize the loop.

We are now at 0.66% of peak. A 60× improvement over Python, but still a factor of 150× from what the hardware can do.

## PyPy (JIT-compiled Python)

**Time: ~12 seconds. Performance: ~175 MFLOPS.**

PyPy uses tracing JIT compilation to optimize Python bytecode. It achieves roughly Java-level performance on pure numerical code. But the same caveats apply: no vectorization, bounds checks on every access.

## C with `-O3`

```rust
for i in 0..n {
    for j in 0..n {
        for k in 0..n {
            c[i][j] += a[i][k] * b[k][j];
        }
    }
}
```

Compiled with `gcc -O3`:

**Time: ~9 seconds. Performance: ~238 MFLOPS.**

The C compiler produces native code directly, without bounds checks or JIT warmup. But the loop order is catastrophically bad for caches. `B[k][j]` accesses memory with stride n, which means each access is to a different cache line. The inner loop is memory-bound, not compute-bound. The CPU spends most of its time waiting for memory.

This is a critical lesson: **native languages give you control, not automatic performance**. The C version is faster than Python, but it's still orders of magnitude from the machine's capability, because the naive implementation ignores the memory hierarchy.

## C with `-O3 -march=native -ffast-math`

With architecture-specific optimization and relaxed floating-point semantics:

**Time: ~0.6 seconds. Performance: ~3.6 GFLOPS.**

The compiler now uses AVX2 instructions (8 floats at a time), reorders floating-point operations, and leverages the CPU's full instruction set. This is roughly 11% of peak — respectable for a simple compilation of the naive triple loop.

## C with Cache Optimization (Loop Reordering)

```rust
for i in 0..n {
    for j in 0..n {
        for k in 0..n {
            c[i][j] += a[i][k] * b[k][j];
        }
    }
}
```

Swapping the j and k loops makes the inner loop access `C[i][j]` and `B[k][j]` sequentially — both are streaming through contiguous memory. The compiler can vectorize this.

Combined with `-march=native -ffast-math`:

**Time: ~0.18 seconds. Performance: ~12 GFLOPS.**

A 3.3× improvement over the naive loop order, just from fixing the memory access pattern. We are now at 37% of peak.

## BLAS (OpenBLAS)

```rust
unsafe {
    cblas_sgemm(
        CblasRowMajor, CblasNoTrans, CblasNoTrans,
        n, n, n, 1.0, A.as_ptr(), n as i32, B.as_ptr(), n as i32, 0.0, C.as_mut_ptr(), n as i32,
    );
}
```

**Time: ~0.12 seconds. Performance: ~18 GFLOPS.**

OpenBLAS uses hand-tuned assembly kernels with cache blocking, register tiling, prefetching, and architecture-specific micro-optimizations. It achieves 56% of the CPU's theoretical peak.

The full case study on matrix multiplication (`algorithms/matmul`) shows how to go from the naive triple loop to ~90% of BLAS performance in under 40 lines of C. The techniques — transposition, blocking, vectorization, register reuse — are the same techniques that apply to every algorithm in this book.

## The Lesson

| Version | Time (s) | MFLOPS | % of Peak |
|---------|----------|--------|-----------|
| Pure Python | 630 | 3.4 | 0.01% |
| PyPy | 12 | 175 | 0.55% |
| Java | 10 | 210 | 0.66% |
| C `-O3` | 9 | 238 | 0.74% |
| C `-march=native -ffast-math` | 0.6 | 3,600 | 11% |
| C + cache-friendly | 0.18 | 12,000 | 37% |
| OpenBLAS | 0.12 | 18,000 | 56% |

The fastest version is 5,250× faster than the slowest. The hardware is the same. The operation count is the same. The difference is entirely in how the code maps to the machine.

This is the central thesis of the book: **performance is not determined by the algorithm's operation count, but by how effectively the implementation exploits the hardware's capabilities.** Understanding that gap is what separates the 3.4 MFLOPS programmer from the 18 GFLOPS engineer.
