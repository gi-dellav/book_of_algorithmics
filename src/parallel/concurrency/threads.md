# Threads

Threads are the default concurrency primitive in most programming languages. Unlike processes, threads share an address space — they can read and write the same variables without IPC. This makes communication cheap but introduces race conditions. The operating system schedules threads preemptively: a thread can be interrupted at any instruction, and another thread takes its place.

## pthreads (POSIX Threads)

The lowest-level thread API on Linux:

```rust
use std::thread;

const N: usize = 1000000;
static mut A: [i32; N] = [0; N];  // Shared array

fn sum_half(start: usize) -> i64 {
    let mut sum: i64 = 0;
    unsafe {
        for i in start..start + N/2 {
            sum += A[i] as i64;
        }
    }
    sum
}

fn main() {
    // Initialize the array
    for i in 0..N { unsafe { A[i] = (i + 1) as i32; } }

    let t1 = thread::spawn(|| sum_half(0));
    let t2 = thread::spawn(|| sum_half(N/2));

    let sum1 = t1.join().unwrap();
    let sum2 = t2.join().unwrap();

    println!("Sum = {}", sum1 + sum2);
    // Expected: N*(N+1)/2 = 500000500000
}
```

Each thread sums half the array. No synchronization needed — each thread writes to its own local `sum`, and the main thread combines after both finish. This is the ideal case for threading: independent work with no shared mutable state.

Thread creation: ~10 µs (vs. ~500 µs for `fork()`). The kernel allocates a stack (~8 KB initially, growing on demand), a task_struct, and sets up the scheduling context. The stack is the main memory cost: 8 MB default virtual size (only committed pages are actually allocated).

## Thread Pools

Creating and destroying threads per task is wasteful. A **thread pool** pre-creates a fixed number of worker threads and dispatches tasks to them via a queue:

```rust
use std::sync::{Arc, Mutex, Condvar};
use std::collections::VecDeque;

// Simplified thread pool (conceptual)
struct ThreadPool {
    workers: Vec<std::thread::JoinHandle<()>>,
    queue: Arc<(Mutex<VecDeque<Task>>, Condvar)>,
}

impl ThreadPool {
    fn new(num_workers: usize) -> Self {
        let queue = Arc::new((Mutex::new(VecDeque::new()), Condvar::new()));
        let workers = (0..num_workers).map(|_| {
            let q = queue.clone();
            std::thread::spawn(move || {
                loop {
                    let (lock, cvar) = &*q;
                    let mut q_lock = lock.lock().unwrap();
                    while q_lock.is_empty() {
                        q_lock = cvar.wait(q_lock).unwrap();
                    }
                    let task = q_lock.pop_front().unwrap();
                    drop(q_lock);
                    execute(task);
                }
            })
        }).collect();
        ThreadPool { workers, queue }
    }
}
```

Thread pools amortize creation cost and bound the number of threads. The optimal number of workers is typically `num_cores` for CPU-bound work and `num_cores / (1 - wait_fraction)` for IO-bound work (where wait_fraction is the fraction of time spent waiting for IO).

## The GIL Problem (Python)

Python's Global Interpreter Lock (GIL) prevents multiple native threads from executing Python bytecode simultaneously. Even on a 64-core machine, only one Python thread runs at a time. The GIL exists because Python's memory management (reference counting) is not thread-safe.

```python
import threading

counter = 0

def increment():
    global counter
    for _ in range(1000000):
        counter += 1  # This is NOT atomic; GIL doesn't protect individual operations

t1 = threading.Thread(target=increment)
t2 = threading.Thread(target=increment)
t1.start(); t2.start()
t1.join(); t2.join()

print(counter)  # NOT 2000000 — race condition even with GIL!
```

The GIL is released during I/O operations and inside C extensions that explicitly release it. For CPU-bound parallelism in Python, use `multiprocessing` (separate processes, each with its own GIL) or C extensions that release the GIL during computation.

## Over-Subscription

Creating more threads than CPU cores is called over-subscription. The OS preempts threads and time-slices between them. Each context switch costs ~2 µs (save/restore registers, potentially TLB flush). If you have 1000 threads on 8 cores, each thread gets ~0.8% of CPU time, and context switch overhead can consume 10-20% of total CPU.

The rule: for CPU-bound work, `num_threads = num_cores`. For IO-bound work, `num_threads = num_cores / (1 - wait_fraction)`. If threads spend 90% of time waiting for disk, `num_threads = num_cores / 0.1 = 10 × num_cores`. Beyond this, additional threads add scheduling overhead without improving throughput.

## Performance (Zen 2, 8 cores)

| Operation | Time |
|-----------|------|
| Thread creation | ~10 µs |
| Thread join | ~5 µs |
| Context switch (same process) | ~2 µs |
| Uncontended mutex lock/unlock | ~25 ns |
| Contended mutex (thread switch) | ~2000 ns |
| Atomic increment | ~7 ns |
| False sharing (two cores writing adjacent cache lines) | ~50 ns per write |

## Key Lessons

1. **Threads share memory by default.** This is both their strength and their weakness. Communication is fast (no IPC overhead) but race conditions are pervasive.
2. **Thread pools are essential for throughput.** Creating threads per task adds ~10 µs overhead. A pool reduces this to the cost of queuing a task (~50 ns with a lock-free queue).
3. **Don't over-subscribe CPU-bound threads.** More threads than cores means context switching overhead. `num_cores` is the sweet spot.
4. **Python threads don't parallelize CPU work.** The GIL serializes bytecode execution. Use `multiprocessing` or C extensions.
5. **False sharing is the silent performance killer.** Two threads writing to different variables in the same cache line (64 bytes) cause constant cache coherence traffic, reducing throughput by 10-100×.
