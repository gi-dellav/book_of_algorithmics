# Benchmarking Methodology

A benchmark is a measurement of performance under controlled conditions. A good benchmark is reproducible, representative, and isolates the effect you're studying. A bad benchmark is worse than no benchmark — it gives you confidence in wrong conclusions.

## The Essential Loop

Benchmarking is not a one-time activity — it's a cycle:

1. **Hypothesize**: "I think optimizing the hash function will improve throughput by 20%."
2. **Measure baseline**: Run the current code under controlled conditions.
3. **Implement change**: Make the optimization.
4. **Measure new version**: Same conditions, same inputs.
5. **Compare**: Did performance improve? By how much? Is the difference statistically significant?
6. **Learn**: Update your mental model. Was the hypothesis correct? Why or why not?

The loop should be fast. If it takes 5 minutes to run a benchmark, you'll run it 3 times and stop. If it takes 5 seconds, you'll run it 100 times, experiment with parameters, and discover effects you didn't anticipate. **Invest in making your benchmarks fast.**

## Setting Up a Benchmark

A minimal Rust benchmark (using Criterion style):

```rust
use std::hint::black_box;

fn bm_sum_array(arr: &[i32]) {
    let mut sum = 0;
    for &val in arr {
        sum += val;
    }
    black_box(sum);  // Prevent elimination
}
```

Key elements:
- **`DoNotOptimize`**: Ensures the compiler doesn't eliminate the computed result. Defined as `asm volatile("" : "+r"(x))`.
- **`SetItemsProcessed`**: Reports throughput (items/second), not just time.
- **`Range`**: Tests multiple input sizes. Performance often changes nonlinearly with size (cache effects).
- **Multiple iterations**: The framework runs enough iterations to get stable timing (typically millions for microbenchmarks, fewer for large ones).

## Splitting Implementation from Benchmark

Put the code under test in a separate `.rs` file from the benchmark harness. Compile each with `-O2` but link them *without LTO* (or use `#[inline(never)]`). This prevents the compiler from optimizing across the boundary and eliminating the code you're trying to measure.

```makefile
# Benchmark Makefile
CXXFLAGS = -O2 -march=native

bench: benchmark.o implementation.o
    $(CXX) $(LDFLAGS) -o bench benchmark.o implementation.o

benchmark.o: benchmark.cpp
    $(CXX) $(CXXFLAGS) -c benchmark.cpp

implementation.o: implementation.cpp
    $(CXX) $(CXXFLAGS) -c implementation.cpp
```

## Analysis with Jupyter

For serious benchmarking, export results to CSV and analyze in a notebook:

```python
import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv("bench_results.csv")
plt.errorbar(df["size"], df["mean_ns"], yerr=df["std_ns"], label="optimized")
plt.xscale("log")
plt.xlabel("Input size")
plt.ylabel("Time (ns)")
plt.legend()
```

The notebook approach:
- Saves you from manually plotting results after each change.
- Makes it easy to compare multiple versions on one graph.
- Records the analysis alongside the data (reproducibility).

## Reporting Results

Always report:
- **Hardware**: CPU model, clock speed, cache sizes, memory configuration.
- **Software**: OS version, compiler version, compiler flags.
- **Benchmark setup**: Input sizes, number of iterations, warm-up procedure.
- **Statistics**: Mean, median, standard deviation, and the number of trials.

Without this information, your numbers are meaningless to anyone else (and to future you).
