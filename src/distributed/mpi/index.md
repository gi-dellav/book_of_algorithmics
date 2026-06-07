# Message Passing (MPI)

MPI is the assembly language of distributed computing: low-level, portable, and ubiquitous. Every supercomputer and cluster runs MPI. An MPI program spawns many processes (one per core or per node) that communicate via `MPI_Send` and `MPI_Recv`. There is no shared memory — all data is explicitly passed in messages. This forces the programmer to think about data distribution from the start.

## Hello MPI

```rust
use mpi::traits::*;

fn main() {
    let universe = mpi::initialize().unwrap();
    let world = universe.world();
    
    let rank = world.rank();  // My process ID
    let size = world.size();  // Total number of processes
    
    println!("Hello from process {} of {}", rank, size);
}
```

Compile with `mpicc`, run with `mpirun -np 8 ./hello`. Each process gets a unique `rank` (0 to size-1). `MPI_COMM_WORLD` is the default communicator containing all processes.

## Point-to-Point Communication

```rust
if rank == 0 {
    let data = 42i32;
    world.process_at_rank(1).send(&data);
    //                           ^dest
} else if rank == 1 {
    let (data, _status) = world.process_at_rank(0).receive::<i32>();
    //                              ^src
    println!("Received: {}", data);
}
```

`MPI_Send` blocks until the send buffer can be reused (implementation-dependent — may wait for matching receive, or may buffer locally). `MPI_Recv` blocks until a matching message arrives. The tag differentiates message types between the same pair of processes.

**Non-blocking variants** enable overlap of communication and computation:

```rust
let (request, _status) = world.process_at_rank(1).immediate_send_with_tag(&data, 0);
// ... do useful work while data is being sent ...
request.wait();  // Ensure send is complete
```

`MPI_Isend` returns immediately. The `request` handle tracks progress. `MPI_Wait` blocks until the operation completes. Between `Isend` and `Wait`, the program can compute — hiding network latency.

## Collective Operations

Collective operations involve all processes in a communicator. They're optimized by the MPI implementation (using tree or ring topologies depending on message size and process count).

### Broadcast

```rust
let mut data: i32 = 0;
if rank == 0 { data = 42; }
world.process_at_rank(0).broadcast_into(&mut data);
// All processes now have data == 42
```

Tree-based broadcast: O(log P) messages instead of P-1. MPI chooses the tree topology automatically.

### Reduce

```rust
let local_sum = compute_local();
let mut global_sum = 0i32;
world.process_at_rank(0).reduce_into(&local_sum, &mut global_sum, SystemOperation::sum());
// global_sum is valid only on process 0
```

Tree-based reduction: processes combine results in pairs, propagating up the tree. Total messages: O(log P). Total data moved: O(P) (each element summed at each level). For large arrays, latency is O(log P) but bandwidth is O(n) per level.

`MPI_Allreduce` stores the result on all processes (extra broadcast at the top of the tree).

### Scatter and Gather

```rust
// Only on process 0
let mut local_chunk = vec![0i32; CHUNK];
world.process_at_rank(0).scatter_into(&mut local_chunk);
world.process_at_rank(0).gather_into(&local_chunk);
```

### Barrier

```rust
world.barrier();
```

Barriers are useful for separating phases in iterative algorithms, but a slow process holds everyone back (the "straggler problem").

## The α-β Model

Point-to-point communication cost for a message of n bytes:

**T(n) = α + β · n**

where:
- **α** (latency): fixed overhead per message (~1 µs for shared memory, ~10 µs for InfiniBand, ~100 µs for Ethernet).
- **β** (inverse bandwidth): time per byte (~0.1 ns/byte for InfiniBand, ~1 ns/byte for Ethernet).

For small messages (n ≤ 1 KB): latency dominates. For large messages (n ≥ 1 MB): bandwidth dominates.

## Distributed Sum with MPI

Summing 1 billion integers across 8 nodes:

```rust
let mut local_sum: i64 = 0;
for i in (rank..N).step_by(size as usize) {  // Round-robin assignment
    local_sum += array[i as usize];
}

let mut global_sum: i64 = 0;
world.all_reduce_into(&local_sum, &mut global_sum, SystemOperation::sum());
```

Each process sums its portion locally (N/P elements, perfectly parallel). Then `MPI_Allreduce` combines the P results in O(log P) time. For N = 10⁹: local sum ~10 ms, reduction ~1 µs. Speedup: ~8× on 8 nodes (near-perfect).

## Performance Tuning

1. **Message size threshold**: small messages (< 1 KB) → optimize for latency; large messages (> 1 MB) → optimize for bandwidth.
2. **Overlap communication and computation**: always use `Isend`/`Irecv` with `Wait` to hide network latency behind useful work.
3. **Collective operations are optimized**: use `MPI_Reduce`/`MPI_Bcast` instead of manual send/recv loops. The MPI implementation uses algorithms tuned for your hardware.
4. **Process-to-core mapping**: `mpirun --bind-to core` prevents process migration. On NUMA machines, use `--map-by numa` to keep processes local to their memory.
5. **Aggregate messages**: send one 1 MB message instead of 1000 1 KB messages. The α term (latency) dominates for small messages.

## Key Lessons

1. **MPI is the primitive, not the framework.** You build distributed algorithms on top of `Send`/`Recv`. MPI provides the communication; you provide the data distribution and synchronization.
2. **The α-β model guides algorithm design.** Minimize the number of messages (α) for small data; minimize total bytes sent (β) for large data.
3. **Collective operations are not magic — they're well-implemented trees.** `MPI_Allreduce` for P processes takes O(log P) messages, not O(P). Trust the implementation.
4. **Non-blocking communication is essential for performance.** `Isend`/`Irecv` with overlap turns a sequential communication-compute loop into a pipelined one. The difference is 2× for many algorithms.
5. **MPI is low-level but portable.** The same MPI program runs on a shared-memory workstation (communication via shared memory) and a 10,000-node cluster (communication via InfiniBand). The MPI implementation handles the difference.
