# External Sorting

Sorting data that doesn't fit in memory requires a different approach from quicksort or mergesort. External sorting minimizes I/O, which dominates runtime when data resides on disk.

## The Problem

N records, M records fit in memory, B records per block (disk page). Sort the N records into order.

Internal sorting: O(N log N) comparisons, but assumes random access is O(1). When N ≫ M, random access costs one I/O per access — not O(1). A standard quicksort would do Θ(N) random accesses, each costing one disk seek (10 ms). For N = 10⁹ (a few GB), that's months of I/O.

External merge sort solves this: O((N/B) log_{M/B}(N/B)) I/Os, which for realistic parameters is 2–3 passes over the data.

## K-Way Merge Sort

### Phase 1: Create Sorted Runs

Read chunks of M records into memory, sort each with an internal sort (quicksort, mergesort), and write out sorted "runs" of size M. Cost: 2 × N/B I/Os (read once, write once).

Number of runs: R = ⌈N/M⌉.

### Phase 2: Merge Runs

If R ≤ M/B (the number of runs fits in memory, with one block per run as a buffer), we can merge all runs in one pass: read one block from each run, find the minimum, output it, refill the block when exhausted.

Cost per merge pass: 2 × N/B I/Os (read all data, write merged result).

Number of passes: ⌈log_{M/B}(R)⌉ ≈ ⌈log_{M/B}(N/M)⌉.

Total I/Os: 2 × N/B × (1 + ⌈log_{M/B}(N/M)⌉).

### Example

- N = 10⁹ records (4 GB of 4-byte ints)
- M = 10⁷ records (40 MB — fits in RAM with room to spare)
- B = 256 records (4 KB pages)
- R = 100 runs
- M/B = 40,000 (we can merge up to ~40,000 runs in one pass, but we only have 100)
- Passes: 1 (create runs) + 1 (merge 100 runs) = 2 passes
- Total I/O: 4 × N/B = 4 × 4 GB / 4 KB = 4 million I/Os

At 10 ms per random I/O (HDD), 4 million random I/Os would take 11 hours. But external merge sort does *sequential* I/O — ~100 MB/s — so 16 GB of I/O takes ~160 seconds. The difference between random and sequential I/O is the difference between 11 hours and 3 minutes.

## Replacement Selection

An improved run-generation algorithm produces runs that are approximately 2M long (on average) rather than exactly M:

1. Load M records into a heap.
2. Repeatedly: extract min → write to output → read next record from input.
3. If the new record ≥ the last output, insert into heap (it belongs to the current run).
4. If the new record < the last output, hold it aside (it belongs to the next run).
5. When the heap is empty, start a new run with the held-aside records.

Because the heap produces sorted output and some input records happen to be larger than the current output, runs grow to ~2M on average. Fewer runs → fewer merge passes.

## Practical Implementation

```rust
use std::io::{Read, Write};

// K-way merge using a tournament tree (or heap)
struct HeapEntry {
    value: i32,
    run_idx: usize,
}

fn kway_merge<R: Read, W: Write>(
    input_runs: &mut [R],
    output: &mut W,
    _total_records: usize,
) {
    let k = input_runs.len();
    let mut heap: Vec<HeapEntry> = Vec::with_capacity(k);
    let mut buf = [0u8; 4];

    // Initialize heap with first record from each run
    for i in 0..k {
        if input_runs[i].read_exact(&mut buf).is_ok() {
            heap.push(HeapEntry {
                value: i32::from_ne_bytes(buf),
                run_idx: i,
            });
        }
    }
    build_min_heap(&mut heap);

    // Extract minimum, replace from the same run
    while !heap.is_empty() {
        let min_value = heap[0].value;
        let min_run_idx = heap[0].run_idx;
        output.write_all(&min_value.to_ne_bytes()).unwrap();

        if input_runs[min_run_idx].read_exact(&mut buf).is_ok() {
            heap[0].value = i32::from_ne_bytes(buf);
            heap[0].run_idx = min_run_idx;
            heapify_down(&mut heap, 0);
        } else {
            // This run is exhausted — replace with last element
            heap.swap_remove(0);
            heapify_down(&mut heap, 0);
        }
    }
}
```

The heap operations are O(log k) each. For k = 100 (typical), this is ~7 comparisons per record — negligible compared to the I/O cost.

**Optimization**: Use a tournament tree instead of a heap for the k-way merge. Tournament trees do fewer comparisons per element (log₂ k instead of 2 log₂ k) but have more complex update logic. The difference matters when comparisons are expensive (e.g., long string keys) but is minor for integers.

## Hash Join vs. Sort-Merge Join

When joining two large tables on a key, two approaches:

**Sort-Merge Join**: Sort both tables on the join key, then merge. I/O: sort cost for both tables + one sequential scan.

**Hash Join**: Partition both tables by hash of the join key into M/B buckets. Each bucket fits in memory. Join each bucket with an in-memory hash table. I/O: 3 passes (partition + build + probe) for each table.

In practice, hash join is faster when the hash table fits in memory and partitioning overhead is low. Sort-merge wins when the data is already sorted (or needs to be sorted for other reasons) or when there are many duplicate keys (the merge step is linear).

External sorting is one of the most-studied problems in computer science, and the algorithms used in databases are highly optimized. But the core principle — minimize random I/O by doing sequential passes — appears throughout this book.
