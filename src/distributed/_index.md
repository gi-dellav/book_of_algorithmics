# Computation Across Machines

Distributed computing is parallelism at the next scale. When your problem doesn't fit on one machine — or when one machine isn't fast enough, or when you need fault tolerance — you distribute across many machines connected by a network. The rules change: communication costs dominate, failures are inevitable, and consistency becomes a design choice rather than a given.

## Why Distributed Computing Is Different

A single machine provides:
- **Shared memory**: all threads see the same data. Latency: ~100ns (L3 miss to RAM).
- **Reliable hardware**: CPUs don't crash (usually). Memory errors are rare (though not zero).
- **Synchronous clocks**: all cores share a timebase. Operations happen in a well-defined order.

A distributed system has:
- **Message passing**: data moves via network. Latency: ~10 µs (InfiniBand), ~100 µs (Ethernet), ~100 ms (cross-continent).
- **Unreliable components**: machines crash, networks partition, disks fail. The system must tolerate partial failure.
- **No global clock**: each machine has its own clock. Event ordering requires explicit protocols (Lamport timestamps, vector clocks).

The network is 1000-100,000× slower than main memory. This changes everything: algorithms that were memory-bound on a single machine become communication-bound in a distributed system. The goal shifts from "minimize memory accesses" to "minimize messages."

## The Distributed Algorithm Design Space

| Concern | Single Machine | Distributed |
|---------|---------------|-------------|
| Communication | Memory bus (100 GB/s) | Network (1-100 Gb/s) |
| Latency | ~100 ns (RAM) | ~10 µs (InfiniBand) to 100 ms (WAN) |
| Synchronization | Mutexes, atomics | Consensus (Paxos/Raft), atomic broadcast |
| Failure model | Rare (hardware errors) | Common (nodes, network, disks) |
| Consistency | Sequential consistency | Eventual, causal, strong |
| Scaling limit | ~256 cores, ~4 TB RAM | ~10,000 nodes, ~1 PB aggregate RAM |

## The CAP Theorem (Brewer, 2000)

A distributed system can provide at most two of three guarantees:
- **Consistency**: every read sees the latest write (or an error).
- **Availability**: every request receives a (non-error) response, even during partitions.
- **Partition tolerance**: the system continues to function despite network partitions.

Proof: consider two nodes separated by a network partition. A write arrives at node A. Node A can either accept the write (sacrificing consistency — node B won't see it) or reject it (sacrificing availability). Partition tolerance is non-negotiable in practice (networks fail), so the real choice is: consistency or availability during a partition?

CP systems (ZooKeeper, etcd, HBase): sacrifice availability during partitions. All nodes agree on state; if a majority is unreachable, the system blocks.

AP systems (Cassandra, DynamoDB, Riak): sacrifice consistency during partitions. All nodes accept writes; conflicts are resolved later (last-write-wins, vector clocks, CRDTs).

## The 8 Fallacies of Distributed Computing (Deutsch, 1994)

1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.

Every distributed algorithm makes assumptions about which fallacies it can tolerate. TCP handles #1 (packet loss) but not #3 (bandwidth limits). Consensus handles #1 and #5 (node failures) but requires majority quorum.

## What This Chapter Covers

1. **Message Passing (MPI)** — The lowest-level distributed primitive: `MPI_Send` and `MPI_Recv`. Collective operations (broadcast, reduce, gather). Performance models for latency and bandwidth.
2. **Distributed Algorithms** — Sorting (sample sort), matrix multiplication (Cannon's algorithm, SUMMA), and graph processing (parallel BFS, PageRank). How to adapt single-machine algorithms to the distributed setting.
3. **MapReduce** — The programming model that democratized distributed computing. Map, shuffle, reduce. The Hadoop/Spark ecosystem. When MapReduce works and when it doesn't.
4. **The Actor Model** — Erlang/Akka approach: actors with mailboxes, no shared state, explicit message passing. Fault tolerance through supervision.
5. **Cloud Computing** — The infrastructure layer: virtual machines, containers, orchestration (Kubernetes), and serverless. The economics of renting vs. owning.

## Prerequisites

This chapter assumes familiarity with the earlier parallel computing chapter. The concurrency models (threads, fibers, event loops) and synchronization primitives (mutexes, atomics) are the foundation. Distributed computing adds the network and failure model on top.

The hardware perspective from earlier chapters still applies: each node is a full machine with its own memory hierarchy, caches, and SIMD units. A distributed algorithm runs on many such machines. Optimizing the single-node performance (chapters 1-13) amplifies the distributed performance — a 10× faster single-node implementation means 10× fewer nodes needed.
