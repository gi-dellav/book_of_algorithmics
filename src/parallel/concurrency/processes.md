# Processes

Processes are the oldest concurrency primitive. A process is an independent program with its own memory space, file descriptors, and scheduling context. Two processes share nothing by default — communication requires explicit IPC (pipes, shared memory, sockets). This isolation is both a strength (no race conditions on shared data) and a weakness (communication overhead).

## fork() and exec()

On Unix, a new process is created with `fork()`. The child inherits a copy-on-write duplicate of the parent's memory. `exec()` replaces the process image with a new program.

```rust
use std::process::Command;

fn main() {
    let child = Command::new("ls")
        .arg("-l")
        .spawn()
        .expect("fork/exec failed");

    let output = child.wait_with_output().expect("wait failed");
    println!("Child exited with status {}", output.status);
}
```

`fork()` takes ~0.5 ms on a modern Linux system (page table copy, not memory copy — the memory is CoW-mapped). `exec()` takes ~1 ms (load binary, set up address space). Total process creation: ~1.5 ms. Compare with thread creation: ~10 µs. Processes are ~150× heavier to create.

## Shared Memory

Since processes have separate address spaces, sharing data requires explicit setup:

```rust
use std::process::Command;
use std::ptr;

fn main() {
    unsafe {
        // Create a shared memory segment via mmap (using libc)
        let shared = libc::mmap(
            ptr::null_mut(),
            std::mem::size_of::<i32>() * 1024,
            libc::PROT_READ | libc::PROT_WRITE,
            libc::MAP_SHARED | libc::MAP_ANONYMOUS,
            -1,
            0,
        ) as *mut i32;

        match unsafe { libc::fork() } {
            0 => {
                // Child writes to shared memory
                *shared = 42;
                libc::_exit(0);
            }
            pid => {
                libc::waitpid(pid, ptr::null_mut(), 0);
                println!("Child wrote: {}", *shared);
                libc::munmap(shared as *mut libc::c_void, std::mem::size_of::<i32>() * 1024);
            }
        }
    }
}
```

`MAP_SHARED` ensures both processes see the same physical pages. Without it (`MAP_PRIVATE`), the child gets a CoW copy and changes are invisible to the parent.

Shared memory is fast: ~100ns for a read (same as any memory access). But it reintroduces race conditions. Mutexes don't work across processes (they're in different address spaces). Instead, use **POSIX semaphores** or **futex** shared between processes:

```rust
// POSIX named semaphores (using libc)
unsafe {
    let sem = libc::sem_open(
        b"/my_semaphore\0".as_ptr() as *const i8,
        libc::O_CREAT,
        0o644,
        1,
    );
    libc::sem_wait(sem);    // Lock
    *shared = 42;
    libc::sem_post(sem);    // Unlock
    libc::sem_close(sem);
    libc::sem_unlink(b"/my_semaphore\0".as_ptr() as *const i8);
}
```

## When to Use Processes

**Strengths:**
- Fault isolation: a crash in one process doesn't affect others. Chrome uses a process per tab — one tab crashing doesn't bring down the browser.
- Security isolation: different processes can run as different users, with different capabilities.
- Language/runtime independence: each process can be written in a different language.
- Implicit parallelism: no shared state → no race conditions (by default).

**Weaknesses:**
- Creation cost: ~1.5 ms vs. ~10 µs for threads.
- Communication overhead: IPC is slower than shared memory access (pipes: ~1 µs/message, sockets: ~5 µs/message).
- Memory overhead: each process has its own page tables (~50 KB minimum) and address space overhead.
- Scheduling overhead: context switch between processes is more expensive than between threads (TLB flush on some architectures, though PCID mitigates this on x86).

**Best for:**
- Multi-tenant servers (one process per request provides fault isolation).
- CPU-bound workloads that don't need frequent communication (embarrassingly parallel: process each file in a directory independently).
- Security-sensitive applications (process isolation is a security boundary; thread isolation is not).
- Chrome-style multi-process architectures.

**Avoid when:**
- You need microsecond-latency communication between workers.
- You need to share large data structures (copying or IPC is expensive).
- You're creating and destroying workers frequently (use a pool or switch to threads).

## Performance (Zen 2, Linux 5.x)

| Operation | Time |
|-----------|------|
| fork() | ~500 µs |
| fork() + exec() | ~1500 µs |
| Process context switch | ~2 µs |
| Pipe read/write (4 bytes) | ~0.8 µs |
| Shared memory read (64 bytes) | ~0.05 µs |
| POSIX semaphore lock/unlock | ~0.3 µs |
