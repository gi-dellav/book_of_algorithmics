# Interrupts and Syscalls

The CPU does not run your program in isolation. Periodically, it must stop what it's doing to handle external events: a keystroke, a network packet, a timer tick, or a request from your program to the operating system. These interruptions are the bridge between your code and the world outside the CPU core.

## What Is an Interrupt?

An **interrupt** is a signal that causes the CPU to suspend its current instruction stream and transfer control to a special handler routine. After the handler completes, the CPU resumes the interrupted code as if nothing happened.

Sources of interrupts:
- **Hardware interrupts**: A device (keyboard, disk, NIC, timer) signals the CPU via an interrupt controller.
- **Software interrupts**: Your program executes a special instruction (`int`, `syscall`, `sysenter`) to request OS services.
- **Exceptions**: The CPU detects an error condition (division by zero, page fault, invalid opcode) and raises an exception.

Exceptions are further classified:
- **Faults**: Correctable error; the instruction can be retried after the handler fixes the problem (e.g., page fault — the OS loads the page from disk, then the instruction re-executes transparently).
- **Traps**: Intentional; used for debugging (breakpoints, single-step).
- **Aborts**: Unrecoverable; the program must be terminated (e.g., double fault, machine check).

## The Interrupt Handling Dance

When an interrupt arrives:

1. The CPU finishes or aborts the current instruction.
2. It pushes the current `rip` (return address), `cs` (code segment), and `rflags` (flags register) onto the stack.
3. It looks up the interrupt handler address in the **Interrupt Descriptor Table** (IDT) — an array of gate descriptors indexed by interrupt vector number.
4. It jumps to the handler, possibly switching to a kernel stack and elevating privilege level.
5. The handler saves the remaining registers, does its work, restores registers, and executes `iret` (interrupt return) to pop `rip`, `cs`, and `rflags` from the stack.

The cost: on a modern x86-64 processor, a round-trip interrupt (user → kernel → user) costs roughly **100–300 cycles** for a minimal handler. This is the fundamental reason system calls are more expensive than function calls — a `call` instruction costs ~2 cycles; a `syscall` costs ~100+.

## System Call Mechanisms

There are three ways to make a system call on x86-64 Linux:

### `int 0x80` (Legacy)

```asm
mov eax, 1          ; syscall number: sys_write
mov edi, 1          ; fd = stdout
mov rsi, message    ; buffer
mov edx, 13         ; length
int 0x80            ; invoke
```

This is the old 32-bit interface. It's slow because `int` goes through the full interrupt machinery (IDT lookup, privilege checks, stack switch). Use it only for compatibility with ancient kernels.

### `syscall` (Fast System Call)

```asm
mov rax, 1          ; syscall number
mov rdi, 1          ; fd
mov rsi, message    ; buf
mov rdx, 13         ; count
syscall             ; fast! (but still ~100 cycles)
```

`syscall` was introduced with AMD64. It uses a dedicated mechanism:
- Switches to ring 0 (kernel mode).
- Saves `rip` to `rcx` and `rflags` to `r11`.
- Jumps to the address in the `STAR` (LSTAR) model-specific register (MSR).

No IDT lookup, no stack switch (the kernel uses a separate kernel stack, but the switch is handled by the `syscall`/`sysret` hardware). Much faster than `int 0x80`.

### `sysenter` (Intel's Equivalent)

Intel's counterpart to `syscall`. Used on 32-bit systems; `syscall` is preferred on 64-bit. Similar performance.

### The vDSO

For read-only, non-security-sensitive syscalls like `gettimeofday` and `clock_gettime`, Linux provides the **vDSO** (virtual dynamic shared object) — a small shared library mapped into every process's address space that performs the syscall in userspace by reading from a memory page shared with the kernel. This eliminates the kernel transition entirely for these operations.

```rust
// This may never enter the kernel:
let mut ts = libc::timespec { tv_sec: 0, tv_nsec: 0 };
unsafe { libc::clock_gettime(libc::CLOCK_MONOTONIC, &mut ts) };  // ~15 ns via vDSO
```

## Why Syscalls Are Expensive

Even `syscall`/`sysret`, the fastest path, costs ~100 cycles because:

1. **Mode switch**: The CPU transitions from ring 3 (user) to ring 0 (kernel). This involves privilege checks and loading kernel-mode segment descriptors.
2. **Register save/restore**: The kernel must save all user registers (not just the few `syscall` saves automatically) and restore them before returning.
3. **TLB and cache effects**: The kernel address space is different from user space. Switching address spaces (even with same-CR3 optimization) may cause TLB misses.
4. **Spectre/Meltdown mitigations**: Since 2018, kernel entry/exit includes additional flushes and barriers to prevent speculative execution attacks (KPTI, retpoline). These add 50–200 cycles.
5. **Pipelining disruption**: The pipeline is essentially flushed during the mode switch — all in-flight instructions must complete or be discarded.

For comparison:
- Function call: ~2 cycles (plus stack frame overhead)
- Direct syscall (`syscall`/`sysret`): ~100–300 cycles
- Indirect syscall (via `int 0x80`): ~300–500 cycles
- With PTI (page table isolation): add ~50–300 cycles

## Practical Implications

1. **Batch syscalls**: `read` 64 KB at once, not 1 KB 64 times. The per-call overhead dominates for small operations.

2. **Use `io_uring`**: Linux's `io_uring` subsystem allows submitting multiple I/O requests to a shared ring buffer without entering the kernel each time. The kernel processes them asynchronously. For high-throughput I/O, this eliminates syscall overhead almost entirely.

3. **Avoid syscalls in hot loops**: Moving a file descriptor check or error status check into the inner loop can cost you 100 cycles per iteration. Do checks once outside the loop.

4. **`mmap` vs. `read`**: For large sequential file access, `mmap` maps the file into memory and lets the kernel handle I/O transparently via page faults. Fewer explicit syscalls, but the page fault cost is similar. Benchmark both approaches.

5. **Thread context switches are even worse**: A context switch between threads involves saving/restoring the full register set, switching page tables (sometimes), and scheduler bookkeeping. Cost: 1,000–10,000 cycles.

## Context Switches

A context switch is not a syscall, but it's the other expensive kernel transition you need to understand:

```
Thread A running → interrupt/timer → scheduler runs → Thread B running
```

Cost components:
- Save A's registers.
- Flush TLB (if switching between processes with different address spaces).
- Load B's registers.
- Indirect branch predictor and cache are cold for B.

A context switch costs roughly 1–10 µs (2,000–20,000 cycles at 2 GHz). This is why lock contention hurts: a thread waiting on a contended mutex may be descheduled, and the resulting context switch dwarfs the cost of the mutex operations themselves.

The parallel computing chapter (Chapter 13) covers synchronization, scheduling, and how to avoid context-switch overhead in multi-threaded code.
