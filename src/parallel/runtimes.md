# Threading Runtimes

Threading runtimes are the infrastructure between your code and `pthread_create`. They manage thread pools, work distribution, and synchronization, so you don't write thread management code by hand. The three dominant runtimes in 2024: OpenMP (scientific computing), Intel TBB (C++ libraries), and language-level runtimes (Go, Rust, Erlang).

## OpenMP

OpenMP is a compiler-based runtime. You annotate loops and regions with pragmas; the compiler generates the threading code.

```rust
use rayon::prelude::*;

(0..N).into_par_iter().for_each(|i| {
    a[i] = b[i] + c[i];
});
```

OpenMP maintains a thread pool. On first entering a parallel region, threads are forked. On exit, they spin-wait (not sleep) for the next parallel region. This amortizes thread creation cost across the program's lifetime.

**Scheduling policies**:
- `schedule(static, chunk)`: divide iterations into equal chunks at compile time. Zero runtime overhead but susceptible to load imbalance.
- `schedule(dynamic, chunk)`: threads grab chunks from a shared counter at runtime. Higher overhead but handles imbalance.
- `schedule(guided, chunk)`: chunk size starts large and shrinks. Balances overhead (fewer grabs early) with load balancing (small chunks at the end to fill gaps).

## Intel TBB (Threading Building Blocks)

TBB is a C++ library that provides high-level parallel patterns:

```rust
use rayon::prelude::*;

// Parallel for
(0..N).into_par_iter().for_each(|i| {
    a[i] = b[i] + c[i];
});

// Parallel reduce
let sum = a.par_iter().sum();

// Pipeline
let (tx1, rx1) = std::sync::mpsc::channel();
let (tx2, rx2) = std::sync::mpsc::channel();

std::thread::spawn(move || {
    while let Some(data) = read_next() { tx1.send(data).ok(); }
});
std::thread::spawn(move || {
    for data in rx1 { tx2.send(process(data)).ok(); }
});
for result in rx2 { write_result(result); }
```

TBB uses **work stealing**: each thread has a local deque of tasks. When a thread's deque is empty, it steals from another thread's deque. This is the same algorithm as Go's goroutine scheduler and Cilk (the original work-stealing language from MIT, 1994).

## Cilk / OpenCilk

Cilk (Blumofe & Leiserson, 1994) introduced work stealing. The key concept: **work and span**:
- **Work** T₁: total operations if executed sequentially.
- **Span** T∞: operations on the critical path (dependencies). The minimum possible execution time with infinite processors.

Brent's theorem: execution time on P processors is bounded by `Tₚ ≤ T₁/P + T∞`. The T₁/P term is the parallelizable work; the T∞ term is the inevitable serial bottleneck.

Cilk's `spawn` and `sync` model:

```rust
fn fib(n: i32) -> i32 {
    if n < 2 { return n; }
    let (x, y) = rayon::join(
        || fib(n - 1),  // This can run in parallel
        || fib(n - 2),  // This runs in the current thread
    );
    x + y
}
```

The runtime guarantees: if you have P processors, the execution is within `T₁/P + O(T∞)` of optimal. The work-stealing scheduler is provably efficient.

## Language-Level Runtimes

**Go**: the Go runtime provides goroutines, channels, and the select statement. The scheduler (since 1.14) is fully preemptive — long-running goroutines are preempted at safe points. The runtime also handles `epoll` integration for network I/O.

**Rust (tokio)**: tokio is an async runtime that uses work-stealing task schedulers. It's built on Rust's `async`/`await` and provides a multi-threaded runtime with `#[tokio::main]`. Unlike Go, tokio is opt-in: you choose it (or `async-std`, or `smol`) as a library, not built into the language.

**Erlang/Elixir (BEAM VM)**: the BEAM VM runs Erlang processes (actors) with a preemptive scheduler. Each process gets a reduction budget (~2000 function calls) before being preempted. The scheduler uses per-core run queues with work stealing. Processes are so lightweight (300 bytes initial) that millions can coexist.

## Choosing a Runtime

| Criteria | OpenMP | TBB | Go | Rust/tokio |
|----------|--------|-----|-----|-----------|
| Language | C/C++/Fortran | C++ | Go | Rust |
| Programming model | Pragmas on loops | Template library | Goroutines + channels | async/await + streams |
| Load balancing | Static/dynamic/guided | Work stealing | Work stealing | Work stealing |
| Overhead | Very low (compiler) | Low (library) | Low (runtime) | Low (compiler + library) |
| Debugging | Hard (implicit parallelism) | Medium | Medium | Medium |
| Best for | Numeric loops | General C++ parallelism | Network services | Network services, safety-critical |

## Key Lessons

1. **Work stealing is the universal scheduling algorithm.** Cilk, TBB, Go, tokio, and BEAM all use variants. It's decentralized, scalable, and provably within a constant factor of optimal.
2. **Work and span determine scalability.** If your span (critical path) is long, no amount of processors will help. Optimize the critical path first.
3. **OpenMP is the simplest for loop parallelism.** A single `#pragma omp parallel for` turns a scalar loop into a parallel one. But it's limited to structured parallelism (no pipelining, no complex DAGs).
4. **Language-level runtimes win for IO-bound workloads.** Go and tokio integrate I/O with the scheduler so blocking operations don't block OS threads. OpenMP and TBB can't do this natively.
5. **The runtime choice is less important than the parallel algorithm.** A bad parallel algorithm on the best runtime loses to a good algorithm on a naive runtime. Algorithm first, runtime second.
