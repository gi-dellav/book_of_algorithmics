# MapReduce

MapReduce (Dean & Ghemawat, 2004) is the programming model that democratized distributed computing. Before MapReduce, distributed programming meant MPI — explicit message passing, failure handling, and data distribution. MapReduce abstracted all of that behind two functions: `map` and `reduce`. The runtime handles distribution, failures, and the shuffle in between.

## The Programming Model

```python
# Word count in MapReduce
def map(key, value):
    # key = document ID, value = document text
    for word in value.split():
        emit(word, 1)  # Emit (word, 1) for each word

def reduce(key, values):
    # key = word, values = list of counts [1, 1, 1, ...]
    emit(key, sum(values))
```

That's it. The programmer writes `map` and `reduce`. The MapReduce runtime:
1. Splits input into chunks, distributes to workers.
2. Each worker runs `map` on its chunks, producing intermediate (key, value) pairs.
3. **Shuffle**: partition intermediate pairs by key, send all pairs with the same key to the same reducer.
4. Each worker runs `reduce` on its keys, producing final output.

The shuffle is where MapReduce earns its name: it's an all-to-all communication phase that groups values by key. The network is the bottleneck — terabytes of intermediate data moving between machines.

## The Original Google Implementation (2004)

Key design decisions:
- **Fault tolerance via re-execution**: if a worker fails, re-run its tasks on another machine. No checkpointing, no rollback. Map outputs are stored locally; reduce outputs are written to GFS (Google File System).
- **Data locality**: schedule map tasks on machines that already have the input data (GFS replicates data 3×, so the scheduler has choices).
- **Straggler mitigation**: when a job is near completion, launch backup copies of the remaining tasks. Whichever finishes first wins. This handles slow machines (disk errors, network congestion, competing jobs).
- **Combiner functions**: a "mini-reduce" that runs on map outputs before the shuffle. For word count: pre-sum word counts on the mapper before sending. Reduces shuffle data by orders of magnitude.

## Hadoop and Spark

**Hadoop** (2006): open-source MapReduce. Stores data in HDFS (Hadoop Distributed File System). Map outputs are written to disk, read by reducers — the shuffle is disk-based. This made Hadoop reliable but slow (each MapReduce job reads from and writes to disk multiple times).

**Spark** (2014): in-memory MapReduce. Intermediate data is kept in memory (RDDs: Resilient Distributed Datasets) rather than written to disk. This makes iterative algorithms (machine learning, graph processing) 10-100× faster than Hadoop. Spark extends MapReduce with additional operations: `filter`, `join`, `groupBy`, and the DataFrame API.

```scala
// Spark word count
val textFile = spark.read.textFile("hdfs://input")
val counts = textFile.flatMap(line => line.split(" "))
                     .map(word => (word, 1))
                     .reduceByKey(_ + _)
counts.write.csv("hdfs://output")
```

The same MapReduce pattern, but the `reduceByKey` performs a shuffle (all-to-all by key) and Spark keeps the RDD lineage for fault tolerance (if a partition is lost, recompute it from the original data).

## When MapReduce Works (and When It Doesn't)

**Works well:**
- Embarrassingly parallel map phase (independent per-element processing).
- Associative and commutative reduce (sum, count, max — order doesn't matter).
- One shuffle is acceptable (the algorithm only needs one MapReduce step).

**Doesn't work well:**
- Iterative algorithms (PageRank, k-means, gradient descent): each iteration is a MapReduce job, and each job writes intermediate data to disk (Hadoop). Spark solves this with in-memory RDDs.
- Algorithms with complex data dependencies: MapReduce forces everything into the map-shuffle-reduce pipeline. Graph algorithms, matrix factorizations, and anything with non-trivial communication patterns require multiple MapReduce jobs (each with a shuffle).
- Low-latency queries: MapReduce is batch-oriented. A job takes seconds to minutes to launch. For sub-second queries, use a database, not MapReduce.

## The Shuffle: The Heart of MapReduce

The shuffle is an all-to-all communication: every mapper sends data to every reducer. For M mappers and R reducers: M × R connections. With M = 10,000 and R = 5000: 50 million connections. The network is the bottleneck.

Techniques to optimize the shuffle:
- **Combiner**: reduce data at the mapper before sending. For word count: pre-sum counts → 100× less shuffle data.
- **Partitioning**: hash keys to reducers. If the key distribution is skewed (one key has 90% of values), one reducer gets all the work. Use a custom partitioner if you know the key distribution.
- **Sort-based shuffle**: Hadoop sorts intermediate data by key before sending to reducers. This groups all values for a key together, enabling the reducer to process keys sequentially. The sort is O(n log n) per reducer but enables efficient reduce (streaming merge, no hash table in memory).

## Beyond MapReduce

Modern distributed computing frameworks have moved beyond MapReduce:
- **Spark**: RDDs, DataFrames, and structured streaming. MapReduce is just one pattern among many.
- **Flink**: true streaming (event-at-a-time processing) with exactly-once semantics. MapReduce is batch; Flink is streaming.
- **Ray**: distributed task-based parallelism for Python. More flexible than MapReduce (any function can be a task) but requires more programmer effort for fault tolerance.
- **Dask**: distributed computing for Python with NumPy/Pandas-like APIs. MapReduce under the hood for reductions, but the user writes NumPy code.

MapReduce as a programming model is still relevant — every distributed system has a shuffle phase, whether it's called "reduce," "groupBy," or "aggregate." But MapReduce as a standalone system (Hadoop) has been superseded by more flexible frameworks.

## Key Lessons

1. **MapReduce is the simplest distributed programming model.** Two functions — `map` and `reduce` — are enough to express a surprising range of computations (sorting, indexing, machine learning, graph processing).
2. **The shuffle is the bottleneck.** All-to-all communication is unavoidable for grouping by key. Combiners, efficient serialization, and in-memory shuffle (Spark) mitigate but don't eliminate it.
3. **Fault tolerance via re-execution is practical.** No checkpointing, no distributed snapshots — just re-run failed tasks. This works because map and reduce are deterministic (or idempotent in practice).
4. **MapReduce is batch-oriented. Streaming requires different abstractions.** Flink, Kafka Streams, and Spark Streaming handle continuous data. MapReduce assumes a finite dataset with a beginning and end.
5. **The MapReduce model is alive, even if Hadoop is declining.** Spark, Dask, Ray, and Flink all expose group-by-key operations that are conceptually MapReduce. Understanding the shuffle is transferable.
