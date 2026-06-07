# Atomics and Lock-Free Programming

Atomics are the foundation of lock-free programming. A mutex serializes access to shared data by making threads wait. An atomic operation never blocks — it completes in a single, indivisible hardware operation. The CPU guarantees that no other core observes a partially-completed atomic operation.

## What "Atomic" Means

An atomic operation has two guarantees:
1. **Indivisibility**: no thread observes a partially-executed operation. A 64-bit atomic store on a 32-bit architecture writes all 8 bytes atomically — no thread sees 4 old bytes + 4 new bytes.
2. **Visibility**: with appropriate memory ordering, the result becomes visible to other threads in a well-defined order.

```rust
use std::sync::atomic::{AtomicI32, Ordering};

static COUNTER: AtomicI32 = AtomicI32::new(0);

fn increment() {
    COUNTER.fetch_add(1, Ordering::SeqCst);  // Atomic, lock-free
}
```

`atomic_fetch_add` compiles to `lock xadd` on x86 — a single instruction with the `LOCK` prefix that guarantees atomicity across all cores. Latency: ~7 cycles on Zen 2. Compare with `pthread_mutex_lock` (25 cycles uncontended) — atomics are ~3.5× faster for this simple case.

## The Atomic Operations

C11/C++11 atomics provide a complete set of lock-free primitives:

| Operation | x86 Instruction | Description |
|-----------|----------------|-------------|
| `atomic_load` | `mov` | Read (implicitly atomic for aligned ≤ 64-bit) |
| `atomic_store` | `mov` (+ `mfence` for seq_cst) | Write |
| `atomic_fetch_add` | `lock xadd` | Add and return old value |
| `atomic_fetch_sub` | `lock xadd` (with negated arg) | Subtract and return old value |
| `atomic_exchange` | `xchg` (implicit `lock`) | Swap and return old value |
| `atomic_compare_exchange_strong` | `lock cmpxchg` | CAS: compare and swap if equal |
| `atomic_compare_exchange_weak` | `lock cmpxchg` | CAS that may fail spuriously |
| `atomic_fetch_and` | `lock and` | Bitwise AND |
| `atomic_fetch_or` | `lock or` | Bitwise OR |

The `lock` prefix on x86 locks the cache line for the duration of the instruction — no other core can read or write it. The cost: ~7-20 cycles for the `lock` + arithmetic operation, plus cache coherence traffic.

## Compare-and-Swap (CAS)

CAS is the universal atomic primitive. Given an expected value and a new value, it atomically:
1. Reads the current value.
2. If `current == expected`: write `new`.
3. If `current != expected`: do nothing.
4. Returns the value read (whether the swap happened or not).

```rust
use std::sync::atomic::{AtomicI32, Ordering};

// CAS: if *ptr == expected, set *ptr = desired and return true
fn cas(ptr: &AtomicI32, expected: i32, desired: i32) -> bool {
    ptr.compare_exchange(expected, desired, Ordering::SeqCst, Ordering::SeqCst).is_ok()
}

// Lock-free counter increment using CAS
fn cas_increment(counter: &AtomicI32) {
    let mut old = counter.load(Ordering::SeqCst);
    while counter.compare_exchange_weak(old, old + 1, Ordering::SeqCst, Ordering::SeqCst).is_err() {
        // old is automatically updated to the current value on failure
        // Loop until CAS succeeds
    }
}
```

CAS can implement any atomic operation — `fetch_add` can be written as a CAS loop. But native `fetch_add` is faster because it's a single instruction rather than a loop that may retry multiple times under contention.

## Lock-Free Stack

The simplest lock-free data structure: a LIFO stack using CAS on the head pointer:

```rust
use std::sync::atomic::{AtomicPtr, Ordering};

struct Node {
    value: i32,
    next: *mut Node,
}

static HEAD: AtomicPtr<Node> = AtomicPtr::new(std::ptr::null_mut());

fn push(value: i32) {
    let node = Box::into_raw(Box::new(Node { value, next: std::ptr::null_mut() }));
    loop {
        unsafe { (*node).next = HEAD.load(Ordering::Acquire); }
        if HEAD.compare_exchange_weak(
            unsafe { (*node).next },
            node,
            Ordering::Release,
            Ordering::Acquire,
        ).is_ok() { break; }
    }
}

unsafe fn pop() -> Option<Box<Node>> {
    loop {
        let node = HEAD.load(Ordering::Acquire);
        if node.is_null() { return None; }  // Empty
        let next = (*node).next;
        if HEAD.compare_exchange_weak(node, next, Ordering::Release, Ordering::Acquire).is_ok() {
            return Some(Box::from_raw(node));
        }
    }
}
```

Both `push` and `pop` retry on CAS failure (another thread modified `head` between our load and our CAS). This is the **CAS loop** pattern — the fundamental construct of lock-free programming.

The ABA problem: between reading `node->next` and CAS, another thread pops `node`, pushes a new node that happens to reuse the same address, and makes `node->next` point somewhere else. The CAS succeeds because the pointer matches, but the semantics are wrong. Solutions: use a **tagged pointer** (combine pointer + counter in a 128-bit atomic) or hazard pointers (keep a thread-local list of "in-use" nodes).

## Lock-Free Queue (Michael-Scott)

A lock-free multi-producer multi-consumer queue:

```rust
use std::sync::atomic::{AtomicPtr, Ordering};

struct Node {
    value: i32,
    next: AtomicPtr<Node>,
}

struct Queue {
    head: AtomicPtr<Node>,
    tail: AtomicPtr<Node>,
}

impl Queue {
    fn enqueue(&self, value: i32) {
        let node = Box::into_raw(Box::new(Node {
            value,
            next: AtomicPtr::new(std::ptr::null_mut()),
        }));

        loop {
            let tail = self.tail.load(Ordering::Acquire);
            let next = unsafe { (*tail).next.load(Ordering::Acquire) };

            if tail == self.tail.load(Ordering::Acquire) {  // tail was consistent
                if next.is_null() {
                    // tail is the last node. Try to link our new node.
                    if unsafe { (*tail).next.compare_exchange_weak(
                        next, node, Ordering::Release, Ordering::Acquire
                    ).is_ok() } {
                        // Success! Advance tail (may fail — other enqueuer will do it)
                        let _ = self.tail.compare_exchange(
                            tail, node, Ordering::Release, Ordering::Acquire
                        );
                        return;
                    }
                } else {
                    // tail is behind. Help advance it.
                    let _ = self.tail.compare_exchange(
                        tail, next, Ordering::Release, Ordering::Acquire
                    );
                }
            }
        }
    }
}
```

This is production code (Java's `ConcurrentLinkedQueue`, Rust's `crossbeam::queue`). The key insight: an enqueuer that observes `tail` lagging behind `tail->next` helps advance `tail` before retrying. This ensures progress even if the original enqueuer is descheduled.

## Performance and Scalability

Atomics are fast but don't eliminate contention. If 8 cores all do `atomic_fetch_add` on the same variable, each increment takes ~50 ns (up from 7 ns) — the LOCK prefix serializes access to the cache line. The cache line bounces between cores, each getting exclusive ownership, doing the increment, and releasing.

**False sharing** makes this worse: if two unrelated atomics happen to be in the same cache line, they contend as if they were the same variable. The fix: pad to 64 bytes:

```rust
use std::sync::atomic::AtomicI32;

#[repr(align(64))]  // Ensure this is the only thing in its cache line
struct PaddedAtomic {
    value: AtomicI32,
    _padding: [u8; 60],
}
```

## When to Go Lock-Free

Lock-free data structures are not always faster than mutex-based ones. They win when:
- **Contention is high**: mutex contention causes thread sleeps (~2 µs). CAS loops keep the core busy (~20 ns/retry).
- **Critical sections are short**: if the protected operation is a few instructions, the mutex overhead (~25 ns) dominates. Atomics (7 ns) are cheaper.
- **Latency matters**: mutexes can sleep threads, adding unpredictable latency. Lock-free guarantees bounded progress.

Lock-free structures lose when:
- **Contention is very high**: CAS loops can spin forever (theoretically), burning CPU. A mutex at least parks the thread.
- **Complexity hurts**: the Michael-Scott queue is ~50 lines of tricky CAS logic. A mutex-protected queue is 5 lines, obviously correct, and fast enough for 90% of use cases.

## Performance (Zen 2, 8 threads contending on the same cache line)

| Operation | Uncontended | Contended (8 threads) |
|-----------|-------------|----------------------|
| `atomic_fetch_add` | 7 ns | 50 ns |
| CAS (success) | 10 ns | 80 ns |
| CAS loop (1 retry avg) | 15 ns | 120 ns |
| Mutex lock+unlock | 25 ns | 2000 ns |
| Lock-free stack push | 20 ns | 150 ns |
| Lock-free queue enqueue | 40 ns | 250 ns |

## Key Lessons

1. **Atomics are 3-5× faster than mutexes for simple operations.** The `lock` prefix is cheaper than the futex infrastructure. Use atomics when the operation maps to a single atomic instruction.
2. **CAS is the universal primitive.** Any lock-free data structure can be built from CAS + retry loops. But native atomic operations (`fetch_add`, `exchange`) are faster than CAS loops.
3. **Contention is the enemy, not the synchronization mechanism.** Both atomics and mutexes degrade under high contention. The real fix is to reduce sharing: shard data, use per-thread counters, batch operations.
4. **Lock-free doesn't mean wait-free.** A CAS loop can retry indefinitely if other threads keep modifying the variable. Wait-free algorithms (each thread completes in a bounded number of steps) are even harder to design.
5. **The ABA problem is real.** Any lock-free data structure that uses CAS on pointers must handle the case where a pointer is reused. Tagged pointers or hazard pointers are the standard solutions.
