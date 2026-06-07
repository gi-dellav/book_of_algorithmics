# Fibers and Goroutines

Threads are scheduled by the OS kernel. A context switch involves a syscall, register save/restore, and potentially a TLB flush — ~2 µs on modern hardware. For workloads with millions of concurrent tasks (handling HTTP requests, actors in a simulation), this overhead is unacceptable. Fibers and goroutines move scheduling to user space, where context switches cost ~10-50 ns — 100× faster.

## What Are Fibers?

A fiber (or coroutine) is a user-space thread. The OS doesn't know fibers exist — it sees a single thread. The runtime scheduler multiplexes many fibers onto a few OS threads. When a fiber blocks (e.g., waiting for I/O), the runtime switches to another fiber without involving the kernel.

This is **M:N scheduling**: M fibers are multiplexed onto N OS threads. The runtime manages fiber creation, scheduling, and context switching, all in user space.

```c
// Conceptual fiber API (not a real library)
fiber_t f1 = fiber_create(my_func, arg);
fiber_t f2 = fiber_create(other_func, arg2);
fiber_yield();  // Give up the CPU, let another fiber run
fiber_join(f1);
```

Each fiber has its own stack (typically 4-16 KB, compared to 8 MB for a thread). Stack switching is just a pointer swap: save `rsp`, load the next fiber's `rsp`. This is ~10 CPU instructions — no syscall, no kernel involvement.

## Goroutines (Go)

Go's goroutines are the most visible fiber implementation. A goroutine is a lightweight coroutine managed by the Go runtime:

```go
func worker(id int, jobs <-chan int, results chan<- int) {
    for j := range jobs {
        results <- j * 2  // Some work
    }
}

func main() {
    const numJobs = 1000000
    jobs := make(chan int, numJobs)
    results := make(chan int, numJobs)
    
    // Start 8 worker goroutines
    for w := 0; w < 8; w++ {
        go worker(w, jobs, results)
    }
    
    // Send jobs
    for j := 0; j < numJobs; j++ {
        jobs <- j
    }
    close(jobs)
    
    // Collect results
    for j := 0; j < numJobs; j++ {
        <-results
    }
}
```

A goroutine starts with a 2 KB stack that grows as needed. The Go runtime's scheduler uses **work stealing**: when an OS thread runs out of goroutines, it steals half the queue from another thread. This balances load automatically without centralized coordination.

Goroutine creation: ~2 µs (vs. ~10 µs for a thread). Goroutine context switch: ~50 ns (vs. ~2 µs for a thread). You can comfortably run 100,000 concurrent goroutines in a Go process — something impossible with OS threads (memory alone: 100,000 × 8 MB = 800 GB of stack space).

## Stackful vs. Stackless Coroutines

**Stackful coroutines** (fibers, goroutines, Lua coroutines): each coroutine has its own stack. You can yield from anywhere — inside a nested function call, inside a loop — and the stack is preserved. The cost: allocated stack space per coroutine (though usually small and growable).

**Stackless coroutines** (Rust `async`/`await`, C++20 coroutines, Python `async`): the compiler transforms the function into a state machine. Each suspend point (`await`) is a state transition. No separate stack is needed — the state machine is stored in a fixed-size struct.

```rust
async fn fetch_url(url: &str) -> Result<String, Error> {
    let response = reqwest::get(url).await?;  // Suspend here
    let body = response.text().await?;        // Suspend here
    Ok(body)
}

// Desugars roughly to:
enum FetchUrlState {
    Start,
    AfterGet { /* saved locals */ },
    AfterText { /* saved locals */ },
    Done,
}

struct FetchUrlFuture {
    state: FetchUrlState,
    url: String,
    // Saved locals...
}
```

Stackless coroutines have zero per-coroutine memory overhead beyond the state machine (typically 32-256 bytes). They can't be suspended from within a nested function call unless that function is also `async` — a limitation known as "function coloring." But for I/O-heavy workloads (web servers, database drivers), the tradeoff is worth it.

## N:M Scheduling (Go Runtime)

Go's scheduler (since Go 1.14) is a fully preemptive work-stealing scheduler:

1. **G**: goroutine (the task).
2. **M**: machine (an OS thread).
3. **P**: processor (a logical CPU, typically `GOMAXPROCS` = number of cores).

Each P has a local run queue of goroutines. An M must acquire a P to execute goroutines. When an M blocks (syscall), it releases its P so another M can use it.

Work stealing: when a P's queue is empty, it randomly selects another P and steals half its queue. This is provably efficient — the expected number of steals is O(log n) for n goroutines.

Network polling: Go registers all network file descriptors with `epoll` (Linux) or `kqueue` (macOS). When a goroutine blocks on I/O, the runtime unparks it when the fd is ready. This is transparent to the programmer — blocking I/O looks like synchronous code but is implemented asynchronously.

## Performance (Zen 2, 8 cores)

| Operation | OS Thread (pthread) | Goroutine | C++20 Coroutine |
|-----------|---------------------|-----------|-----------------|
| Creation | 10 µs | 2 µs | 0.05 µs (stackless) |
| Context switch | 2 µs | 0.05 µs | 0.01 µs |
| Memory per task | 8 MB (virtual) | 2 KB (initial) | ~100 B (state) |
| Concurrent tasks (16 GB RAM) | ~2,000 | ~1,000,000 | ~10,000,000+ |

## Key Lessons

1. **User-space scheduling is 100× faster than kernel scheduling.** No syscalls, no TLB flushes, no register save/restore beyond what's needed.
2. **Fibers/goroutines make massive concurrency practical.** Running 100,000 concurrent HTTP handlers is routine in Go. Doing the same with threads would exhaust memory.
3. **Stackless coroutines are lighter but have the coloring problem.** Rust's `async`/`await` is extremely efficient but requires all async functions to be called with `.await`. This permeates the entire call chain.
4. **Work stealing is the standard scheduling algorithm.** It's decentralized, scalable, and provably efficient. Go, Erlang, Intel TBB, and Cilk all use variants of it.
5. **The OS thread is still the unit of parallelism.** M:N scheduling requires at least one OS thread per CPU core. The I/O is still done by the kernel (via epoll/kqueue). Fibers are an abstraction, not a replacement for the kernel's scheduler.
