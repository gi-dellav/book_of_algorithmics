# Instrumentation

Instrumentation is the simplest profiling technique: you insert measurement code into your program and record how long sections take. It's coarse but trustworthy — you're measuring exactly what you think you're measuring.

## Wall Clock Time

The simplest measurement:

```rust
use std::time::Instant;

let start = Instant::now();
do_work();
let elapsed = start.elapsed();
```

Caveats:
- `clock()` measures CPU time, not wall-clock time. If your process sleeps or is descheduled, `clock()` doesn't count it. For wall-clock, use `clock_gettime(CLOCK_MONOTONIC, &ts)`.
- Resolution: typically 1 µs to 1 ms, depending on the OS and hardware. For shorter operations, run them in a loop.

```rust
use std::time::Instant;

let start = Instant::now();
do_work();
let elapsed = start.elapsed();
let ns = elapsed.as_nanos();
```

## Cycle-Accurate Timing

For microbenchmarks of very short operations, `rdtsc` (Read Time-Stamp Counter) gives cycle-level resolution:

```rust
#[cfg(target_arch = "x86_64")]
use std::arch::x86_64::{__rdtsc, _mm_lfence};

unsafe {
    _mm_lfence();  // Serialize — ensure previous instructions complete before rdtsc
    let start = __rdtsc();
    do_work();
    _mm_lfence();  // Serialize — ensure do_work completes before next rdtsc
    let end = __rdtsc();
    let cycles = end - start;
}
```

Critical details:
- `__rdtsc()` returns a 64-bit cycle counter. On modern CPUs, it counts at the base (non-turbo) frequency. If the CPU is in turbo mode, the counter ticks at the base rate.
- `_mm_lfence()` serializes the instruction stream. Without it, the CPU may execute `rdtsc` before `do_work` completes, or execute the first `rdtsc` while previous instructions are still in flight.
- `rdtscp` is an alternative that is partially serializing and also reads the processor ID.

For most purposes, `clock_gettime` with a loop is preferable to `rdtsc` — it's portable, doesn't require serialization, and measures wall-clock time which is what users care about. Use `rdtsc` only when you need cycle-level precision for operations under ~100 ns.

## The Geometric Distribution Trick

When profiling individual operations (e.g., "how long does one hash table lookup take?"), the measurement overhead can dominate. The trick: sample randomly, not every iteration.

```rust
// Instead of measuring every iteration:
for i in 0..N {
    start_timer();
    hash_lookup(&table, keys[i]);
    stop_timer();
}

// Sample with geometric distribution:
let mut next_sample = 0;
for i in 0..N {
    if i == next_sample {
        start_timer();
        hash_lookup(&table, keys[i]);
        stop_timer();
        // Schedule next sample: p = 0.01 → average 100 iterations between samples
        next_sample = i + geometric_random(0.01);
    } else {
        hash_lookup(&table, keys[i]);  // No measurement overhead
    }
}
```

The timer overhead is spread across ~100 operations, making it negligible. The tradeoff: you need more total iterations to get the same number of samples.

## Instrumentation Best Practices

1. **Don't instrument inside the innermost loop.** The timer call itself may be more expensive than the code you're measuring.
2. **Use a monotonic clock.** `CLOCK_MONOTONIC` is not affected by NTP adjustments or daylight saving.
3. **Warm up before measuring.** The first execution may include cold caches, lazy library initialization, or JIT compilation.
4. **Run multiple trials and report statistics.** A single run is subject to noise. Report median, mean, and standard deviation (or min/max for latency measurements where outliers matter).
5. **Prevent compiler optimization of the measured code.** If the compiler can prove the result is unused, it may eliminate the computation entirely. Use `__asm__ volatile("" : "+r"(result))` or `benchmark::DoNotOptimize(result)` to force the computation to be emitted.
