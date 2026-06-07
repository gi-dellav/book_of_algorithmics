# Noise and Bias in Benchmarking

Modern computers are non-deterministic. The same code on the same input can vary by 10–30% in execution time due to factors beyond your control. Understanding and controlling for these sources of noise is the difference between a real measurement and a random number.

## Sources of Noise

### 1. CPU Frequency Scaling

Modern CPUs vary their clock frequency dynamically (DVFS — Dynamic Voltage and Frequency Scaling). A core might run at 1.5 GHz one moment and 4.0 GHz the next, depending on temperature, power budget, and workload.

- **Turbo Boost**: Short bursts of high frequency for single-thread workloads.
- **Thermal throttling**: Frequency reduction when the CPU gets too hot.
- **Power limits**: Sustained AVX-512 workloads may cause frequency reduction.

**Fix**: Disable frequency scaling during benchmarks:
```bash
sudo cpupower frequency-set --governor performance
```
Or pin to a fixed frequency:
```bash
sudo cpupower frequency-set -f 2.0GHz
```

### 2. Address Space Layout Randomization (ASLR)

ASLR randomizes the location of the stack, heap, and libraries. Different runs may produce different cache alignment, TLB behavior, and branch predictor state due to different memory layouts.

**Fix**: Disable ASLR for benchmarks:
```bash
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```
Or run the benchmark many times to average over randomization effects.

### 3. Executable Name Effects

The path and name of the executable affect its memory layout (the ELF interpreter uses the path). Two identical binaries placed at different paths may have different performance because their code sections map to different cache sets.

**Fix**: Run all versions from the same path (rename the binary if comparing versions).

### 4. Cold vs. Warm Caches

The first run of a benchmark fills the cache. Subsequent runs hit the cache. Reporting the first-run time (cold cache) vs. the steady-state time (warm cache) answers different questions:
- **Cold**: "How fast does the program start?" — relevant for interactive applications.
- **Warm**: "How fast is the inner loop?" — relevant for throughput-oriented code.

**Fix**: Decide which you're measuring. For warm-cache, run a few warm-up iterations before timing. For cold-cache, flush caches with a large `memset` or by running a cache-thrashing program between trials.

### 5. Context Switches and Interrupts

The OS may schedule another thread on your core, evict your caches and TLB entries, and steal cycles. A single unfortunate context switch during a microbenchmark can inflate the measurement by 10,000×.

**Fix**:
- Pin the benchmark to a specific core: `taskset -c 0 ./benchmark`.
- Isolate that core from the OS scheduler: `isolcpus=0` in the kernel boot parameters.
- Run with real-time priority: `chrt --rr 99 ./benchmark`.
- Discard outliers: report median and 95th percentile, not mean.

### 6. Over-Optimization by the Compiler

The compiler may eliminate the code you're trying to measure because it can prove the result is unused.

```c
int sum = 0;
for (int i = 0; i < n; i++)
    sum += a[i];
// If 'sum' is never used, the compiler may delete the entire loop!
```

**Fix**: Use `benchmark::DoNotOptimize(sum)` or `asm volatile("" : "+r"(sum))` to consume the result.

### 7. Measurement Overhead

The timer call itself consumes cycles. For microbenchmarks of single-digit nanosecond operations, the timer overhead dominates.

**Fix**: Measure many iterations in one timing window. Never put a timer inside the innermost loop.

### 8. Statistical Significance

Two runs of the same benchmark on the same machine with the same binary will differ. The question: is the difference between version A and version B *larger* than the run-to-run variance?

**Fix**: Run at least 10 trials. Perform a statistical test (t-test, Mann-Whitney U). Report confidence intervals. If the improvement is 2% and the stddev is 3%, the result is noise.

## Measuring Latency vs. Throughput

**Latency**: Time to complete one operation. Measured by timing a single operation (or a batch, divided by batch size), ensuring no overlap.

**Throughput**: Rate of completing operations. Measured by running many independent operations concurrently and dividing total work by total time.

The difference matters because pipelining and parallelism may give high throughput with high latency, or low latency with low throughput. A CPU can issue one floating-point division every 13 cycles (throughput), but each division takes 13 cycles (latency). They're the same for division because it's not pipelined. For multiplication, throughput is 0.5 cycles but latency is 3 cycles.

To measure latency, introduce a data dependency that prevents the CPU from overlapping operations:
```c
for (int i = 0; i < n; i++)
    x = compute(x, a[i]);  // x depends on previous iteration
```

To measure throughput, ensure independence:
```c
for (int i = 0; i < n; i++)
    results[i] = compute(a[i], b[i]);  // Each iteration is independent
```

## A Benchmarking Checklist

Before trusting a benchmark:
- [ ] CPU frequency locked to a known value.
- [ ] ASLR disabled or averaged over.
- [ ] Warm-up iterations run before timing.
- [ ] Benchmark pinned to a specific core.
- [ ] Result consumed (`DoNotOptimize`).
- [ ] Multiple trials performed.
- [ ] Mean, median, and stddev reported.
- [ ] Source code and build flags documented.
- [ ] Results reproducible on a second machine (or at least a second run).

If all items are checked, your numbers are trustworthy. If not, you're measuring noise.
