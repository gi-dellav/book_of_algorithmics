# Mutual Exclusion

Mutual exclusion is the fundamental synchronization primitive: ensure that only one thread executes a critical section at a time. Without it, concurrent access to shared data produces race conditions — corrupt data, lost updates, and crashes that only manifest under load.

## The Problem: Race Conditions

```rust
static mut COUNTER: i32 = 0;  // Shared

unsafe fn increment() {
    COUNTER += 1;  // Compiles to: mov eax, [COUNTER]; inc eax; mov [COUNTER], eax
}
```

Two threads calling `increment()` can interleave:
```
Thread A: mov eax, [counter]   // eax = 0
Thread B: mov eax, [counter]   // eax = 0
Thread A: inc eax              // eax = 1
Thread B: inc eax              // eax = 1
Thread A: mov [counter], eax   // counter = 1
Thread B: mov [counter], eax   // counter = 1  ← Should be 2!
```

The fix: wrap `counter++` in a critical section protected by a mutex.

## Mutex (pthreads)

```rust
use std::sync::Mutex;
use std::sync::Arc;
use std::thread;

let mutex = Arc::new(Mutex::new(0));

fn increment(m: Arc<Mutex<i32>>) {
    for _ in 0..1000000 {
        let mut counter = m.lock().unwrap();
        *counter += 1;
    }
}
```

The mutex guarantees: at most one thread is inside the critical section at any time. If Thread B calls `lock` while Thread A holds the mutex, Thread B blocks (the OS puts it to sleep) until Thread A calls `unlock`.

## Mutex Internals

On Linux, `pthread_mutex_t` is built on the **futex** (fast userspace mutex) system call. The fast path (no contention) executes entirely in user space:

```rust
use std::sync::atomic::{AtomicI32, Ordering};
use std::arch::asm;

// Simplified futex-based mutex (not the actual implementation)
unsafe fn mutex_lock(lock: &AtomicI32) {
    loop {
        // Fast path: try to acquire with an atomic compare-and-swap
        if lock.compare_exchange(0, 1, Ordering::Acquire, Ordering::Relaxed).is_ok() {
            return;  // Acquired without kernel involvement
        }
        
        // Slow path: the mutex is held. Wait on the futex.
        // The FUTEX_WAIT syscall tells the kernel to sleep this thread
        // until another thread calls FUTEX_WAKE on this address.
        if lock.load(Ordering::Relaxed) == 1 {
            unsafe {
                let addr = lock as *const AtomicI32 as *const i32;
                libc::syscall(libc::SYS_futex, addr, libc::FUTEX_WAIT, 1, 0, 0, 0);
            }
        }
        // Loop back and try again
    }
}

unsafe fn mutex_unlock(lock: &AtomicI32) {
    lock.store(0, Ordering::Release);
    // Wake one waiting thread (if any)
    let addr = lock as *const AtomicI32 as *const i32;
    libc::syscall(libc::SYS_futex, addr, libc::FUTEX_WAKE, 1, 0, 0, 0);
}
```

The atomic compare-and-swap (`lock cmpxchg`) on x86 takes ~8 cycles. If uncontended, lock+unlock is ~25 ns — no syscall, just two atomic operations. If contended, the syscall adds ~2 µs (context switch to another thread).

## Spinlocks

A spinlock is a mutex that never sleeps — it repeatedly checks the lock in a busy loop:

```rust
use std::sync::atomic::{AtomicI32, Ordering};
use std::hint;

fn spinlock_lock(lock: &AtomicI32) {
    while lock.swap(1, Ordering::Acquire) == 1 {
        // Spin: repeatedly check
        // On x86: 'pause' instruction to avoid memory order mis-speculation
        hint::spin_loop();
    }
}

fn spinlock_unlock(lock: &AtomicI32) {
    lock.store(0, Ordering::Release);
}
```

Spinlocks are faster than mutexes for very short critical sections (< 100 ns): no syscall overhead, and the thread stays scheduled on the core. But they waste CPU while spinning. The kernel uses spinlocks internally (where sleeping is not an option). User-space code should use mutexes unless the critical section is under ~100 cycles and contention is rare.

The `PAUSE` instruction on x86 is crucial: it tells the CPU that this is a spin-wait loop, reducing power consumption and avoiding memory order mis-speculation penalties when the lock is released.

## Deadlocks

Two threads, each holding a lock the other needs:

```rust
// Thread A                          // Thread B
let _l1 = mutex1.lock().unwrap();     let _l2 = mutex2.lock().unwrap();
let _l2 = mutex2.lock().unwrap();     let _l1 = mutex1.lock().unwrap();
// Deadlock: A waits for mutex2, B waits for mutex1
```

Prevention strategies:
- **Lock ordering**: always acquire locks in the same order (e.g., always `mutex1` before `mutex2`).
- **`try_lock`**: attempt to lock; if it fails, release all held locks and retry.
- **Deadlock detection**: the OS or runtime detects cycles in the wait-for graph and kills one thread.

## Reader-Writer Locks

A read-write lock allows multiple concurrent readers but only one writer:

```rust
use std::sync::RwLock;

let rwlock = RwLock::new(0);

// Reader
let val = *rwlock.read().unwrap();

// Writer
*rwlock.write().unwrap() = new_val;
```

Performance: uncontended rdlock is ~20 ns; wrlock is ~25 ns. But the real cost is contention — many readers can starve a writer (if a new reader arrives before the previous reader finishes, the writer never gets the lock). Some implementations offer writer-preference variants.

RW locks are useful for read-heavy workloads (configuration data, caches, routing tables). But the overhead is higher than a simple mutex, so for short critical sections, a mutex may be faster even if the access is read-only.

## Performance (Zen 2, 8 threads)

| Operation | Uncontended | Contended (8 threads) |
|-----------|-------------|----------------------|
| `pthread_mutex_lock+unlock` | 25 ns | 2000 ns |
| `atomic_fetch_add` | 7 ns | ~50 ns |
| Spinlock (lock+unlock) | 10 ns | 500 ns (spinning) |
| `pthread_rwlock_rdlock+unlock` | 20 ns | 80 ns |
| `pthread_rwlock_wrlock+unlock` | 25 ns | 2000 ns |

## When to Use What

- **Mutex**: default choice. Simple, correct, handles contention gracefully (sleeps).
- **Spinlock**: only for very short critical sections (< 100 ns) under low contention. Used in OS kernels and lock-free data structure fallbacks.
- **RW lock**: read-mostly workloads where the critical section is long enough to justify the overhead (> 50 ns). For very short reads, a mutex is faster.
- **No lock (atomics)**: when the operation can be expressed as a single atomic instruction (increment, swap, CAS). Always faster than a mutex.

## Key Lessons

1. **Uncontended mutexes are fast (25 ns).** The futex mechanism avoids syscalls in the common case. The overhead is two atomic operations.
2. **Contention kills performance.** A contended mutex involves a context switch (~2 µs) — 100× slower than the fast path. Reduce contention by shrinking critical sections or sharding data.
3. **Lock ordering prevents deadlocks.** Consistent acquisition order is the simplest and most effective deadlock prevention technique.
4. **Spinlocks are niche.** Only appropriate when the critical section is shorter than the context switch cost (~2 µs) and contention is rare. The `PAUSE` instruction is mandatory.
5. **RW locks have higher overhead than mutexes.** For short critical sections, the fast mutex beats the slower RW lock even for read-only access. Profile before choosing.
