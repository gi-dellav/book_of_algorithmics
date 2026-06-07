# Processes

Processes are the oldest concurrency primitive. A process is an independent program with its own memory space, file descriptors, and scheduling context. Two processes share nothing by default — communication requires explicit IPC (pipes, shared memory, sockets). This isolation is both a strength (no race conditions on shared data) and a weakness (communication overhead).

## fork() and exec()

On Unix, a new process is created with `fork()`. The child inherits a copy-on-write duplicate of the parent's memory. `exec()` replaces the process image with a new program.

```c
#include <unistd.h>
#include <sys/wait.h>

int main() {
    pid_t pid = fork();
    
    if (pid == 0) {
        // Child process
        printf("Child: PID = %d\n", getpid());
        execl("/bin/ls", "ls", "-l", NULL);  // Replace with 'ls -l'
        // Never reaches here if execl succeeds
    } else if (pid > 0) {
        // Parent process
        int status;
        waitpid(pid, &status, 0);  // Wait for child to finish
        printf("Child exited with status %d\n", WEXITSTATUS(status));
    } else {
        perror("fork failed");
    }
}
```

`fork()` takes ~0.5 ms on a modern Linux system (page table copy, not memory copy — the memory is CoW-mapped). `exec()` takes ~1 ms (load binary, set up address space). Total process creation: ~1.5 ms. Compare with thread creation: ~10 µs. Processes are ~150× heavier to create.

## Shared Memory

Since processes have separate address spaces, sharing data requires explicit setup:

```c
#include <sys/mman.h>
#include <sys/wait.h>

int main() {
    // Create a shared memory segment
    int *shared = mmap(NULL, sizeof(int) * 1024,
                       PROT_READ | PROT_WRITE,
                       MAP_SHARED | MAP_ANONYMOUS, -1, 0);
    
    pid_t pid = fork();
    
    if (pid == 0) {
        // Child writes to shared memory
        shared[0] = 42;
        _exit(0);
    } else {
        waitpid(pid, NULL, 0);
        printf("Child wrote: %d\n", shared[0]);  // Prints 42
        munmap(shared, sizeof(int) * 1024);
    }
}
```

`MAP_SHARED` ensures both processes see the same physical pages. Without it (`MAP_PRIVATE`), the child gets a CoW copy and changes are invisible to the parent.

Shared memory is fast: ~100ns for a read (same as any memory access). But it reintroduces race conditions. Mutexes don't work across processes (they're in different address spaces). Instead, use **POSIX semaphores** or **futex** shared between processes:

```c
#include <semaphore.h>

sem_t *sem = sem_open("/my_semaphore", O_CREAT, 0644, 1);
sem_wait(sem);    // Lock
shared[0] = 42;
sem_post(sem);    // Unlock
sem_close(sem);
sem_unlink("/my_semaphore");
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
