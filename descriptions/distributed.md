# Chapter: Distributed Computing (`distributed/`)

## Overview

Weight 200, marked `draft: true`, part "Distributed Computing." This is essentially an empty scaffold. Every file in this chapter (6 total across 3 subdirectories) contains only YAML frontmatter — a title and weight — with zero content. The structure hints at planned coverage of MPI, distributed algorithms, MapReduce, actor model, and cloud computing.

## Files and Content

| File | Status | Lines | Description |
|------|--------|-------|-------------|
| `_index.md` | Draft | 3 | Section index (`part: Distributed Computing`) |
| `actor.md` | Empty | 3 | Placeholder for Actor Model |
| `cloud.md` | Empty | 3 | Placeholder for Cloud Computing |
| `algorithms/_index.md` | Empty | 3 | Placeholder for Distributed Algorithms |
| `mapreduce/_index.md` | Empty | 3 | Placeholder for MapReduce |
| `mpi/_index.md` | Empty | 3 | Placeholder for Message Passing Interface |

## Strengths

1. **The structure makes sense**: MPI → Distributed Algorithms → MapReduce → Actor Model → Cloud Computing is a logical progression from low-level message passing to high-level cloud abstractions.
2. **Part-level organization**: Using a separate "Part: Distributed Computing" with its own weight (200) correctly positions this as a major section, not just a chapter.

## Areas for Improvement

1. **Completely empty**: Every article is an empty stub. There is zero educational content.
2. **No connection to earlier chapters**: The hardware-focused chapters (architecture, pipelining, SIMD, cache) don't prepare readers for distributed systems concepts. The transition would be jarring.
3. **Missing fundamental topics**: No stubs for: CAP theorem, consensus algorithms (Paxos/Raft), distributed hash tables, fault tolerance, replication, or distributed tracing.

## Recommendations

1. **Write MPI first**: As the lowest-level distributed primitive, MPI is the natural starting point. Cover point-to-point communication, collective operations (broadcast, reduce, gather, scatter), and basic performance models (latency, bandwidth, overlap).
2. **Add algorithms**: Distributed sorting (sample sort, parallel merge), distributed matrix multiplication (Cannon's algorithm, SUMMA), and distributed graph algorithms (parallel BFS, PageRank).
3. **Add MapReduce**: Cover the programming model, Hadoop/Spark architecture, shuffle optimization, and when MapReduce is/isn't appropriate.
4. **Add consensus**: Paxos or Raft explained, with a focus on the performance implications of different failure models.
5. **Consider the scope**: Distributed computing is a vast field. If the book's focus is hardware-level optimization, this part may be too far removed. Consider whether a single survey chapter would be more appropriate than a full multi-chapter part.
6. **Connect to earlier content**: Distributed algorithms still run on real hardware. Discuss how network latency compares to RAM/SSD latency, how batching (analogous to cache lines) matters in messaging, and how SIMD within a node combines with distribution across nodes.
