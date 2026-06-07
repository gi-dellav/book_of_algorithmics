# Statistical Profiling with `perf`

`perf` is Linux's built-in performance analysis tool. It uses hardware performance counters and sampling to tell you exactly where your program spends its time and what the CPU is doing. If you learn one profiling tool, make it `perf`.

## Hardware Performance Counters

Modern CPUs have dozens of built-in counters that track microarchitectural events:

| Event | Description |
|-------|-------------|
| `cycles` | CPU cycles elapsed |
| `instructions` | Instructions retired |
| `cache-references` | Cache accesses |
| `cache-misses` | Cache misses (last-level cache) |
| `branch-instructions` | Branch instructions executed |
| `branch-misses` | Mispredicted branches |
| `L1-dcache-load-misses` | L1 data cache load misses |
| `LLC-load-misses` | Last-level cache load misses |
| `cpu-clock` | Wall-clock time from the CPU's perspective |

The counters are finite — Zen 2 has 6 programmable counters. `perf` multiplexes them when you ask for more events than counters, but multiplexing reduces accuracy.

## `perf stat`: Aggregate Counters

For a quick overview of program behavior:

```bash
perf stat ./program
```

Output:
```
Performance counter stats for './program':
    2,341.23 msec  task-clock         # 0.998 CPUs utilized
        12,345     context-switches   # 5.274 K/sec
           123     cpu-migrations     # 0.053 K/sec
         2,345     page-faults        # 1.002 K/sec
 5,234,567,890     cycles             # 2.237 GHz
 8,123,456,789     instructions       # 1.55 insn per cycle
 1,012,345,678     branches           # 432.567 M/sec
    45,678,901     branch-misses      # 4.51% of all branches
```

Key metrics:
- **IPC** (instructions per cycle): Above 2 is good; below 1 means the CPU is stalling. Zen 2 peak: ~4.
- **Branch miss rate**: Above 2–3% suggests unpredictable branches. Consider branchless alternatives.
- **Context switches**: High rates may indicate thread contention.
- **Page faults**: High rates indicate memory pressure or poor locality.

Specify specific events:

```bash
perf stat -e cycles,instructions,cache-references,cache-misses,L1-dcache-load-misses ./program
```

For a list of available events:
```bash
perf list
```

## `perf record` and `perf report`: Sampling Profiler

To find *where* time is spent:

```bash
perf record ./program         # Record samples (generates perf.data)
perf report                   # Interactive report
```

`perf record` samples the program counter ~4000 times per second (adjustable with `-F`). Each sample records the instruction pointer, the call chain, and optionally additional data. The output is a statistical profile: functions where many samples land are where the program spends most of its time.

In `perf report`:
- Navigate with arrow keys.
- Press `+` to expand a function and see its callees.
- Press `a` to see the annotated assembly (instruction-level heat map).
- Press `/` to search for a function.

**Call graphs** (include function call chains):
```bash
perf record -g ./program
perf report -g graph
```

This shows not just which function is hot, but *who called it*. Essential for understanding context.

## `perf annotate`: Instruction-Level Analysis

For the hottest function in your profile:

```bash
perf annotate hot_function
```

Shows the function's assembly with per-instruction hit counts:
```
       │    xor %eax,%eax
  0.12 │    nop
 38.45 │    mov (%rdi,%rax,4),%ecx    ← 38% of samples land here!
  0.34 │    add %ecx,%edx
  8.21 │    add $0x1,%rax
 52.88 │    cmp %rsi,%rax             ← 53% of samples land here!
       │    jne loop
```

The `mov` instruction is taking 38% of the time — probably a cache miss. The `cmp` + `jne` takes 53% — the branch is predicted, but macro-fusion makes these two instructions appear as one hot spot.

## `perf stat -d`: Deeper Cache Analysis

```bash
perf stat -e cycles,instructions,L1-dcache-loads,L1-dcache-load-misses,LLC-loads,LLC-load-misses ./program
```

If L1 miss rate is high (>5%), your working set is too large for L1. If LLC (L3) miss rate is high, you're hitting RAM frequently. The cache chapter shows how to fix both.

## Common `perf` Commands

```bash
perf top                  # Live sampling (like 'top' for functions)
perf stat -d ./prog       # Detailed counters
perf record -g ./prog     # Record with call graphs
perf report --stdio       # Text-mode report (scriptable)
perf report -g graph      # Call graph visualization
perf annotate funcname    # Instruction-level heat map
perf script               # Dump raw samples (for flame graph scripts)
perf list                 # List available events
perf stat -e 'syscalls:sys_enter_*' ./prog  # Trace syscalls
```

## Flame Graphs

Flame graphs (Brendan Gregg's invention) visualize call stacks:

```bash
perf record -g ./program
perf script | stackcollapse-perf.pl | flamegraph.pl > flamegraph.svg
```

The SVG shows the call stack hierarchy: width = time spent, height = call depth. Hot functions are wide bars; you can click to zoom. Flame graphs are the best tool for *understanding* a complex profile at a glance.

## Limitations

- `perf` requires kernel support (CONFIG_PERF_EVENTS). Most Linux distributions enable it by default.
- Some events require root (`perf stat` with hardware counters may need `sudo` on some configurations; set `kernel.perf_event_paranoid` to 0 or 1).
- `perf` is Linux-specific. On macOS, use `Instruments` (Xcode). On Windows, use VTune or xperf.
- Sampling profilers have statistical error. Functions that run for very short times may not show up. For microbenchmarks, use instrumentation or simulation.
