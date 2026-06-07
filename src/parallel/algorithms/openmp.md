# OpenMP

OpenMP (Open Multi-Processing) is the dominant parallel programming model for shared-memory scientific computing. It's a set of compiler directives (`#pragma omp ...`), library functions, and environment variables supported by GCC, Clang, ICC, and MSVC. The model: fork-join parallelism. The programmer marks parallel regions; the compiler generates the threading code.

## The Core Pragmas

### `parallel for`

Distributes loop iterations across threads:

```rust
use rayon::prelude::*;

(0..N).into_par_iter().for_each(|i| {
    a[i] = b[i] + c[i];
});
```

Each thread executes a contiguous chunk of iterations. The chunk size is N / num_threads by default.

### `parallel`

Creates a team of threads that execute the same block. Combine with `omp_get_thread_num()` for SPMD (Single Program, Multiple Data):

```rust
use std::thread;

let nthreads = thread::available_parallelism().unwrap().get();
thread::scope(|s| {
    for tid in 0..nthreads {
        let chunk = N / nthreads;
        let start = tid * chunk;
        let end = if tid == nthreads - 1 { N } else { start + chunk };
        s.spawn(move || {
            for i in start..end {
                unsafe { a[i] = b[i] + c[i]; }
            }
        });
    }
});
```

### `reduction`

Combines per-thread partial results into a global result:

```rust
use rayon::prelude::*;

let total: i32 = a.par_iter().sum();
```

Supported reduction operators: `+`, `*`, `-`, `&`, `|`, `^`, `&&`, `||`, `min`, `max`. The compiler creates a private copy of `total` per thread, accumulates locally, and combines with the operator at the end. This avoids atomic operations in the inner loop — critical for performance.

### `critical` and `atomic`

Protect shared variables:

```rust
use rayon::prelude::*;
use std::sync::Mutex;

let shared_sum = Mutex::new(0);
let shared_count = Mutex::new(0);

(0..N).into_par_iter().for_each(|i| {
    let result = expensive_computation(i);
    let mut sum = shared_sum.lock().unwrap();
    *sum += result;
    *shared_count.lock().unwrap() += 1;
});
```

`atomic` is lighter — it's a single hardware atomic instruction:

```rust
use std::sync::atomic::{AtomicI32, Ordering};

static SHARED_SUM: AtomicI32 = AtomicI32::new(0);

SHARED_SUM.fetch_add(result, Ordering::Relaxed);  // Compiles to lock add (or lock xadd on x86)
```

`critical` allows arbitrary code in the protected section; `atomic` only allows simple arithmetic/bitwise operations. `atomic` is ~5× faster than `critical` for simple operations.

### `task`

Creates asynchronous tasks that the runtime schedules dynamically:

```rust
fn fib(n: i32) -> i32 {
    if n < 2 { return n; }
    let (x, y) = rayon::join(
        || fib(n - 1),
        || fib(n - 2),
    );
    x + y
}
```

Tasks enable parallelism for irregular workloads (recursion, while loops, graph traversals) where `parallel for` doesn't apply. The runtime uses work stealing to balance tasks across threads.

## Scheduling

The `schedule` clause controls loop iteration distribution:

```rust
use rayon::prelude::*;

// rayon uses work-stealing by default (similar to guided)
(0..N).into_par_iter().for_each(|i| { /* ... */ });

// For explicit chunk size: use par_chunks or chunks()
(0..N).into_par_iter().chunks(16).for_each(|chunk| {
    for i in chunk { /* ... */ }
});

// Or operate on slices directly for static-like scheduling:
a.par_chunks_mut(16).for_each(|chunk| {
    /* ... */
});
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

```rust
use std::sync::Mutex;
use rayon::prelude::*;

let shared_var = Mutex::new(0);  // Shared, protected by Mutex

(0..N).into_par_iter().for_each(|_| {
    let private_var = 0;  // Private: each thread has its own copy
    // Stack variables declared inside the closure are private
});

// firstprivate: clone a value into each thread
let shared_copy = *shared_var.lock().unwrap();
(0..N).into_par_iter().for_each(|i| {
    let local = shared_copy;  // Each thread gets its own copy
});

// lastprivate: capture the logically-last value
let result = Mutex::new(0);
(0..N).into_par_iter().for_each(|i| {
    *result.lock().unwrap() = compute(i);
    // The last thread to execute writes the final value
});
```

## The `nowait` Optimization

By default, `parallel for` and `sections` have an implicit barrier at the end. `nowait` removes it:

```rust
use rayon::prelude::*;

// rayon's parallel iterators use work-stealing.
// Threads that finish a[i] can start b[i] independently.
rayon::join(
    || (0..N).into_par_iter().for_each(|i| a[i] = compute_a(i)),
    || (0..N).into_par_iter().for_each(|i| b[i] = compute_b(i, a[i])),
);
// Without join (sequential ordering), there's an implicit barrier.
```

Without `nowait`: threads finish the first loop, wait at barrier, then start the second. With `nowait`: a thread finishing its first-loop chunk moves immediately to the second loop (processing `b[i]` for `i` where `a[i]` is done). This overlaps independent work.

## Nested Parallelism

```rust
use rayon::prelude::*;

// Nested parallelism (generally not recommended; oversubscribes)
(0..4).into_par_iter().for_each(|i| {
    (0..N).into_par_iter().for_each(|j| {
        // rayon manages total threads globally
    });
});
```

Generally not recommended — the thread pool can oversubscribe. Better: collapse the loops into one:

```rust
use rayon::prelude::*;

// Flatten 2D loops into 1D parallel iterator
(0..M * N).into_par_iter().for_each(|idx| {
    let i = idx / N;
    let j = idx % N;
    a[i][j] = b[i][j] + c[i][j];
});
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
