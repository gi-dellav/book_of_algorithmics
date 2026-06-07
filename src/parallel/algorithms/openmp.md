# OpenMP

OpenMP (Open Multi-Processing) is the dominant parallel programming model for shared-memory scientific computing. It's a set of compiler directives (`#pragma omp ...`), library functions, and environment variables supported by GCC, Clang, ICC, and MSVC. The model: fork-join parallelism. The programmer marks parallel regions; the compiler generates the threading code.

## The Core Pragmas

### `parallel for`

Distributes loop iterations across threads:

```c
#pragma omp parallel for
for (int i = 0; i < N; i++)
    a[i] = b[i] + c[i];
```

Each thread executes a contiguous chunk of iterations. The chunk size is N / num_threads by default.

### `parallel`

Creates a team of threads that execute the same block. Combine with `omp_get_thread_num()` for SPMD (Single Program, Multiple Data):

```c
#pragma omp parallel
{
    int tid = omp_get_thread_num();
    int nthreads = omp_get_num_threads();
    int chunk = N / nthreads;
    int start = tid * chunk;
    int end = (tid == nthreads - 1) ? N : start + chunk;
    for (int i = start; i < end; i++)
        a[i] = b[i] + c[i];
}
```

### `reduction`

Combines per-thread partial results into a global result:

```c
int total = 0;
#pragma omp parallel for reduction(+:total)
for (int i = 0; i < N; i++)
    total += a[i];
```

Supported reduction operators: `+`, `*`, `-`, `&`, `|`, `^`, `&&`, `||`, `min`, `max`. The compiler creates a private copy of `total` per thread, accumulates locally, and combines with the operator at the end. This avoids atomic operations in the inner loop — critical for performance.

### `critical` and `atomic`

Protect shared variables:

```c
#pragma omp parallel for
for (int i = 0; i < N; i++) {
    int result = expensive_computation(i);
    #pragma omp critical
    {
        shared_sum += result;
        shared_count++;
    }
}
```

`atomic` is lighter — it's a single hardware atomic instruction:

```c
#pragma omp atomic
shared_sum += result;  // Compiles to lock add (or lock xadd on x86)
```

`critical` allows arbitrary code in the protected section; `atomic` only allows simple arithmetic/bitwise operations. `atomic` is ~5× faster than `critical` for simple operations.

### `task`

Creates asynchronous tasks that the runtime schedules dynamically:

```c
int fib(int n) {
    if (n < 2) return n;
    int x, y;
    #pragma omp task shared(x)
    x = fib(n-1);
    #pragma omp task shared(y)
    y = fib(n-2);
    #pragma omp taskwait
    return x + y;
}
```

Tasks enable parallelism for irregular workloads (recursion, while loops, graph traversals) where `parallel for` doesn't apply. The runtime uses work stealing to balance tasks across threads.

## Scheduling

The `schedule` clause controls loop iteration distribution:

```c
#pragma omp parallel for schedule(static, 16)
for (int i = 0; i < N; i++) ...

#pragma omp parallel for schedule(dynamic, 16)
for (int i = 0; i < N; i++) ...

#pragma omp parallel for schedule(guided, 16)
for (int i = 0; i < N; i++) ...
```

- **`static[, chunk]`**: iterations divided into chunks of `chunk` size, assigned round-robin to threads at compile time. Zero runtime overhead. Best when all iterations take the same time.
- **`dynamic[, chunk]`**: threads grab `chunk` iterations from a shared counter at runtime. Higher overhead (atomic increment per chunk) but handles load imbalance. Best when iteration times vary unpredictably.
- **`guided[, chunk]`**: chunk size starts large (N / P) and shrinks exponentially to `chunk`. Fewer atomic grabs than dynamic, similar load balancing. The default for many workloads.
- **`auto`**: delegates to the compiler/runtime. Often maps to `static` or `guided`.

Performance (Zen 2, 8 cores, N = 10⁷, varying iteration cost: 10% of iterations take 10× longer):

| Schedule | Time | Efficiency |
|----------|------|------------|
| `static` | 120 ms | 62% (load imbalance) |
| `dynamic, 1` | 95 ms | 78% |
| `dynamic, 64` | 80 ms | 93% |
| `guided, 1` | 78 ms | 95% |
| `guided, 64` | 75 ms | 99% |

## Memory Model

OpenMP threads share memory but have private variables:

```c
int shared_var = 0;  // Shared by default (in C)

#pragma omp parallel
{
    int private_var = 0;  // Private: each thread has its own copy
    // Stack variables declared inside the parallel region are private
}

#pragma omp parallel for firstprivate(shared_var)
for (int i = 0; i < N; i++) {
    // Each thread gets a copy of shared_var, initialized to its value before the loop
}

#pragma omp parallel for lastprivate(result)
for (int i = 0; i < N; i++) {
    result = compute(i);
    // The value from the last iteration (logically) is copied out
}
```

## The `nowait` Optimization

By default, `parallel for` and `sections` have an implicit barrier at the end. `nowait` removes it:

```c
#pragma omp parallel
{
    #pragma omp for nowait
    for (int i = 0; i < N; i++)
        a[i] = compute_a(i);
    
    #pragma omp for
    for (int i = 0; i < N; i++)
        b[i] = compute_b(i, a[i]);  // Depends on a[i]
}
```

Without `nowait`: threads finish the first loop, wait at barrier, then start the second. With `nowait`: a thread finishing its first-loop chunk moves immediately to the second loop (processing `b[i]` for `i` where `a[i]` is done). This overlaps independent work.

## Nested Parallelism

```c
omp_set_nested(1);

#pragma omp parallel for num_threads(4)
for (int i = 0; i < 4; i++) {
    #pragma omp parallel for num_threads(2)
    for (int j = 0; j < N; j++) {
        // 4 × 2 = 8 threads total
    }
}
```

Generally not recommended — the thread pool can oversubscribe. Better: collapse the loops into one:

```c
#pragma omp parallel for collapse(2)
for (int i = 0; i < M; i++)
    for (int j = 0; j < N; j++)
        a[i][j] = b[i][j] + c[i][j];
```

Turns M×N iterations into a single loop of M×N iterations, distributable across all threads.

## Thread Affinity

```bash
export OMP_PROC_BIND=true   # Bind threads to cores (no migration)
export OMP_PLACES=cores     # One thread per physical core
```

Thread migration between cores causes cache misses and NUMA issues. Binding threads to cores improves performance by 5-15% for memory-bound workloads.

## Performance Tuning Checklist

1. **Set `OMP_NUM_THREADS`**: default is all logical cores. For CPU-bound work, use physical cores only (`nproc --all` / 2 on SMT systems).
2. **Choose the right schedule**: `static` for uniform work, `guided` for varying work.
3. **Use `reduction` instead of `atomic`**: reduces contention from N atomics to log(P) combiner steps.
4. **Merge small parallel regions**: Amortize the fork-join overhead (~2 µs on modern OpenMP).
5. **`OMP_PROC_BIND=true`**: always. Thread migration is death for cache locality.
6. **`OMP_PLACES=cores`**: use physical cores, not hyperthreads, for CPU-bound work.

## Key Lessons

1. **OpenMP is low-effort, high-reward for loop parallelism.** Adding `#pragma omp parallel for` to a loop is the cheapest way to get parallelism. But the cost is that all the complexity is hidden — you must understand the scheduling, reduction, and memory model to debug performance.
2. **`reduction` is the most important pragma after `parallel for`.** It eliminates the most common contention pattern in parallel loops. Without it, 90% of parallel loop performance is lost to atomic operations.
3. **Schedule choice matters more than thread count.** A poorly scheduled parallel loop on 16 cores can be slower than a well-scheduled one on 4 cores.
4. **OpenMP can't parallelize everything.** Irregular algorithms (graph traversals, work-queue-based algorithms) need `task` or a different runtime (TBB, Cilk).
5. **Thread affinity is free performance.** `OMP_PROC_BIND=true` should be the default in every deployment.
