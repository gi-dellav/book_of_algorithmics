# Threads

Threads are the default concurrency primitive in most programming languages. Unlike processes, threads share an address space — they can read and write the same variables without IPC. This makes communication cheap but introduces race conditions. The operating system schedules threads preemptively: a thread can be interrupted at any instruction, and another thread takes its place.

## pthreads (POSIX Threads)

The lowest-level thread API on Linux:

```c
#include <pthread.h>
#include <stdio.h>

#define N 1000000
int a[N];  // Shared array

void *sum_half(void *arg) {
    int start = *(int*)arg;
    long long sum = 0;
    for (int i = start; i < start + N/2; i++)
        sum += a[i];
    return (void*)sum;
}

int main() {
    // Initialize the array
    for (int i = 0; i < N; i++) a[i] = i + 1;
    
    pthread_t t1, t2;
    int start1 = 0, start2 = N/2;
    
    pthread_create(&t1, NULL, sum_half, &start1);
    pthread_create(&t2, NULL, sum_half, &start2);
    
    void *sum1, *sum2;
    pthread_join(t1, &sum1);
    pthread_join(t2, &sum2);
    
    printf("Sum = %lld\n", (long long)sum1 + (long long)sum2);
    // Expected: N*(N+1)/2 = 500000500000
}
```

Each thread sums half the array. No synchronization needed — each thread writes to its own local `sum`, and the main thread combines after both finish. This is the ideal case for threading: independent work with no shared mutable state.

Thread creation: ~10 µs (vs. ~500 µs for `fork()`). The kernel allocates a stack (~8 KB initially, growing on demand), a task_struct, and sets up the scheduling context. The stack is the main memory cost: 8 MB default virtual size (only committed pages are actually allocated).

## Thread Pools

Creating and destroying threads per task is wasteful. A **thread pool** pre-creates a fixed number of worker threads and dispatches tasks to them via a queue:

```c
// Simplified thread pool (conceptual)
struct ThreadPool {
    pthread_t *workers;
    int num_workers;
    Task *queue;           // Lock-free or mutex-protected queue
    pthread_mutex_t lock;
    pthread_cond_t cond;   // Signal when work is available
};

void *worker_loop(void *arg) {
    ThreadPool *pool = (ThreadPool*)arg;
    while (true) {
        pthread_mutex_lock(&pool->lock);
        while (pool->queue_empty)
            pthread_cond_wait(&pool->cond, &pool->lock);
        Task t = dequeue(&pool->queue);
        pthread_mutex_unlock(&pool->lock);
        execute(t);
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
