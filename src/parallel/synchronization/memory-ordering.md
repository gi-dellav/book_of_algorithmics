# Memory Ordering

Memory ordering is the most subtle aspect of parallel programming. The question: when Thread A writes to variable X, when does Thread B see that write? The answer depends on the CPU's memory model, the compiler's optimizations, and the memory ordering guarantees you request. Getting this wrong produces bugs that are non-deterministic, hardware-dependent, and nearly impossible to reproduce.

## Why Memory Ordering Exists

CPUs reorder memory operations for performance. A store to memory takes ~200 cycles (if it misses cache). While that store is in flight, the CPU continues executing independent instructions — including loads and stores that appear *after* the pending store in program order.

```rust
static mut X: i32 = 0;
static mut Y: i32 = 0;

// Thread A
unsafe { X = 1; Y = 1; }

// Thread B
unsafe {
    let r1 = Y;
    let r2 = X;
}
```

What values can `(r1, r2)` have? In a sequentially consistent world: `(0,0)`, `(1,0)`, `(0,1)`, or `(1,1)`. But on real hardware, `(0,0)` is also possible — Thread B can see `y = 1` before seeing `x = 1`, even though Thread A wrote `x` first. The CPU reordered Thread A's stores, or Thread B's loads, or both.

## The Memory Ordering Spectrum (C11/C++11)

C11 and C++11 define six memory orderings, from weakest to strongest:

### `memory_order_relaxed`

No ordering guarantees. Only atomicity is guaranteed. The compiler and CPU may reorder this operation freely with respect to other memory operations.

```rust
use std::sync::atomic::{AtomicI32, Ordering};

static X: AtomicI32 = AtomicI32::new(0);
static Y: AtomicI32 = AtomicI32::new(0);

// Thread A
X.store(1, Ordering::Relaxed);
Y.store(1, Ordering::Relaxed);

// Thread B
let r1 = Y.load(Ordering::Relaxed);
let r2 = X.load(Ordering::Relaxed);
// (r1, r2) = (1, 0) is possible — Thread B saw y=1 but x=0
```

Use case: counters where only the total sum matters, not the order of individual increments.

### `memory_order_acquire` / `memory_order_release`

**Acquire**: subsequent loads and stores (in program order) cannot be reordered before this load. This is a "read barrier" — everything after the acquire stays after.

**Release**: preceding loads and stores cannot be reordered after this store. This is a "write barrier" — everything before the release stays before.

Together, acquire-release creates a **happens-before** relationship: if Thread A does a release-store on `x`, and Thread B does an acquire-load on `x` that reads Thread A's stored value, then all writes Thread A did before the release are visible to Thread B after the acquire.

```rust
use std::sync::atomic::{AtomicI32, Ordering};

static READY: AtomicI32 = AtomicI32::new(0);
static mut DATA: i32 = 0;  // NOT atomic

// Thread A
unsafe { DATA = 42; }
READY.store(1, Ordering::Release);  // Release: data = 42 is visible

// Thread B
while READY.load(Ordering::Acquire) == 0 {
    // Spin until ready
}
let r = unsafe { DATA };  // Guaranteed: r == 42
```

This is the standard pattern for passing data between threads. The release on `ready` ensures `data = 42` is visible before `ready = 1`. The acquire on `ready` ensures `data` is read after `ready == 1`.

### `memory_order_acq_rel`

Combines acquire and release. Used for read-modify-write operations (`fetch_add`, `CAS`) where the operation both reads (acquire) and writes (release).

```rust
use std::sync::atomic::{AtomicI32, Ordering};

// Lock-free flag
static FLAG: AtomicI32 = AtomicI32::new(0);

let expected = 0;
if FLAG.compare_exchange(
    expected, 1, Ordering::AcqRel, Ordering::Acquire
).is_ok() {
    // We acquired the flag. All writes from the previous holder are visible.
}
```

### `memory_order_seq_cst` (sequentially consistent)

The strongest ordering. All seq_cst operations appear to execute in a single total order visible to all threads. This is the default for all C11/C++11 atomics without explicit ordering.

```rust
use std::sync::atomic::{AtomicI32, Ordering};

static X: AtomicI32 = AtomicI32::new(0);
static Y: AtomicI32 = AtomicI32::new(0);

// Thread A                            // Thread B
X.store(1, Ordering::SeqCst);           Y.store(1, Ordering::SeqCst);
let r1 = Y.load(Ordering::SeqCst);      let r2 = X.load(Ordering::SeqCst);
```

With seq_cst: `(r1, r2) = (0, 0)` is IMPOSSIBLE. There is a total order of stores and loads. Either Thread A's store to x happens before Thread B's load, or Thread B's store to y happens before Thread A's load, or both.

Seq_cst is the easiest to reason about (matches our sequential intuition) but the most expensive: on x86, it requires an `mfence` instruction after stores (~33 cycles on Zen 2). With acquire-release (which is free on x86 for loads and stores — x86 already provides acquire-release semantics in hardware), no fence is needed.

## x86 Memory Model

x86 has a relatively strong memory model (x86-TSO: Total Store Order):
- **Loads are not reordered with other loads.**
- **Stores are not reordered with other stores.**
- **Stores are not reordered with older loads.** (Loads can be reordered with older stores — this is the famous "store buffer forwarding.")

This means on x86:
- `memory_order_acquire` is free (compiler barrier only, no CPU instruction).
- `memory_order_release` is free (compiler barrier only).
- `memory_order_seq_cst` stores require `mfence` after the store.
- `memory_order_relaxed` is free (no barriers at all).

On ARM (weak memory model), acquire and release are NOT free — they require explicit `dmb` (data memory barrier) instructions. Code that's fast on x86 with acquire-release may be slower on ARM.

## Dekker's Example on x86

```rust
use std::sync::atomic::{AtomicI32, Ordering};

static X: AtomicI32 = AtomicI32::new(0);
static Y: AtomicI32 = AtomicI32::new(0);
static mut R1: i32 = 0;
static mut R2: i32 = 0;

// Thread A                  // Thread B
X.store(1, Ordering::Relaxed);    Y.store(1, Ordering::Relaxed);
unsafe { R1 = Y.load(Ordering::Relaxed); }  unsafe { R2 = X.load(Ordering::Relaxed); }
```

Can `(r1, r2) = (0, 0)` happen on x86? Yes! Even with the strong x86 memory model. The stores `x = 1` and `y = 1` sit in the store buffer before becoming visible. Thread A's load of `y` can happen while `x = 1` is still in the store buffer. Thread B's load of `x` can happen while `y = 1` is still in the store buffer. Both loads see the old value (0).

The fix: make the stores `memory_order_seq_cst` (which adds `mfence` after each store), or use `memory_order_acq_rel` on the loads (which on x86 just requires a compiler barrier — but that's not enough for Dekker; you need `mfence` on the stores).

## Memory Fences (Standalone Barriers)

C11/C++11 also provide standalone fences:

```rust
use std::sync::atomic::{fence, Ordering};

fence(Ordering::Acquire);   // Read barrier
fence(Ordering::Release);   // Write barrier
fence(Ordering::SeqCst);    // Full barrier
```

These affect ALL subsequent or preceding memory operations, not just those on a specific atomic variable. Use cases: integrating with non-atomic code, or when you need ordering between multiple variables.

## The Compiler Barrier

Even without CPU reordering, the compiler can reorder memory operations. The `volatile` keyword is NOT a memory barrier — it only prevents the compiler from optimizing away the access, not from reordering it relative to other accesses.

```rust
static mut DATA: i32 = 0;
static mut READY: i32 = 0;  // volatile does NOT guarantee ordering!

// Thread A
unsafe {
    DATA = 42;
    // Compiler may reorder: ready = 1 before data = 42!
    READY = 1;
}
```

The fix: use `atomic_signal_fence(memory_order_acq_rel)` for compiler-only barriers (no CPU instructions emitted), or use proper atomics.

## Performance (Zen 2)

| Memory Ordering | Load | Store | RMW (CAS/fetch_add) |
|----------------|------|-------|---------------------|
| `relaxed` | ~3 ns | ~3 ns | ~7 ns |
| `acquire`/`release` | ~3 ns (free on x86) | ~3 ns (free on x86) | ~7 ns |
| `seq_cst` | ~3 ns | ~33 ns (*) | ~33 ns (*) |

(*) The `mfence` after a seq_cst store adds ~30 cycles. On x86, seq_cst loads are free (just `mov`); it's the stores that are expensive because of the mandatory fence.

## Key Lessons

1. **Acquire-release is the sweet spot.** It provides the happens-before relationship needed for most lock-free programming patterns, and it's free on x86. Use it as your default.
2. **Seq_cst is for when you need a total order.** It costs an `mfence` on x86 (~33 ns). Most algorithms don't need it. But it's the default — be explicit about ordering if performance matters.
3. **x86 is surprisingly easy. ARM is not.** Code that works on x86 with `relaxed` atomics may fail on ARM. Test on ARM hardware (or use `acquire`/`release` and let the compiler insert the right barriers).
4. **The store buffer is the root of most reordering.** A store sits in the store buffer before becoming globally visible. Loads can bypass the store buffer, seeing "stale" values. This is the hardware optimization that causes Dekker-like surprises.
5. **Memory ordering is a contract, not an implementation.** `memory_order_release` means "I promise that all prior writes are visible." The hardware and compiler work together to fulfill that promise — whether it's free (x86) or requires barriers (ARM) is their problem, not yours.
