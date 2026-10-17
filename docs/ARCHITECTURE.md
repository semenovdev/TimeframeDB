# TimeframeDB — Architecture and Research Direction

> **Status:** Initial architecture proposal (17 October 2026)  
> **Project:** TimeframeDB  
> **Intended model:** Open source, research-driven, Unix-friendly  
> **Document scope:** Architectural decisions, boundaries, design hypotheses, and a validation roadmap. This is not a finalized storage-format or API specification.

## 1. Vision

**TimeframeDB** is an experimental, lightweight, append-oriented time-series database focused on efficient batched ingestion, time-range scans, and aggregation into timeframes.

### Design values

- **Correctness before benchmarks.** Durability, visibility, and time semantics must be explicit and testable.
- **Engineering over language loyalty.** Prefer the implementation that meets correctness, performance, and maintainability goals; Rust is the primary engine language, while Haskell offers an expressive query-language environment.
- **Understandable internals.** Favor a small codebase and explainable algorithms over broad functionality.
- **Unix composition.** Build independently useful programs that communicate via well-defined streams and protocols.
- **Standards where appropriate.** Research novel storage and aggregation algorithms; reuse existing interoperability standards instead of inventing new tabular result formats.
- **Evidence-driven optimization.** Benchmark real implementations, measure resource costs, and retain negative results.

## 2. Initial Workload

The core workload consists of append-oriented, timestamped events such as market ticks or trades. Financial market data is a motivating example, **not** a permanent restriction on the database's domain.

| Operation | Priority | Notes |
| --- | --- | --- |
| Batched, consistent append | Essential | Batches, not individual-row insertion, are the primary write unit. |
| Range scan by event time | Essential | Efficient sequential/batched access to time intervals. |
| Aggregation into timeframes | Essential | Time buckets and OHLCV-like aggregation where the schema permits. |
| Consistent incremental synchronization | Essential | Track source progress, avoid accidental duplicates, recover after failures. |
| Stream results as batches | Essential | Do not materialize an entire result set before delivery. |
| Exact single-record lookup | Low priority | Not an optimization target. |
| Updates, deletes, arbitrary joins | Out of initial scope | Revisit only with a compelling workload. |
| Universal SQL support | Out of initial scope | A limited SQL-like frontend may be added later. |

The initial execution model can assume a **single writer and multiple readers**. Concurrency and durability guarantees must still be specified and tested.

## 3. Process Architecture

TimeframeDB deliberately separates the query frontend and execution engine into independent operating-system processes.

```text
                   Haskell application / CLI
                  +--------------------------+
                  | Typed query DSL          |
                  | (later: SQL-like parser) |
                  | AST + semantic checks    |
                  | Logical Query IR         |
                  +------------+-------------+
                               |
                    versioned query protocol
                               |  stdin / pipe / IPC
                               v
                  +--------------------------+
                  | Rust TimeframeDB engine  |
                  |--------------------------|
                  | Plan validation          |
                  | Physical execution       |
                  | Batched append / scans   |
                  | Timeframe aggregation    |
                  | Storage / index / commit |
                  +------------+-------------+
                               |
                       Arrow IPC Stream
                               |  stdout / pipe / IPC
                               v
             +-------------------------------------+
             | Haskell / Go / Java / Kotlin / C /  |
             | Rust / other Arrow-capable clients |
             | CLI display, transforms, connectors|
             +-------------------------------------+
```

### Decisions

1. **Rust owns storage and execution.** It manages files, indexes, transactions/commits, physical query planning, scans, and aggregates.
2. **Haskell owns the first query frontend.** The first interface is a typed Haskell DSL that builds a logical query representation. A limited SQL-like parser can later target the same semantic model.
3. **No Rust–Haskell FFI is required on the main path.** The boundary is an explicit protocol, not shared native pointers or a shared heap.
4. **Haskell is not an obligatory result intermediary.** Consumers can read the Rust engine's output directly; a Haskell CLI is simply one possible consumer.
5. **Rust must validate incoming plans independently.** A plan received from any client is untrusted input, even if a Haskell frontend has validated it.

The components should remain usable independently. The engine can be driven by handcrafted test plans before the Haskell DSL exists.

## 4. Query Frontends and Logical Plans

### Phase 1: Native Haskell DSL

The first frontend is a library for constructing typed queries. It should expose operations such as:

- Select a data source and optional partition/instrument.
- Restrict an event-time range.
- Read raw events in batches.
- Aggregate into supported timeframes.
- Project or format supported outputs.

Its exact syntax is undecided. The following is **illustrative pseudocode**, not a promised API:

```haskell
query =
  from "trades"
    |> symbol "BTCUSDT"
    |> between start end
    |> ohlcv Minutes5
```

### Phase 2: SQL-like frontend

A deliberately limited SQL-like parser can be implemented in Haskell and compiled into the **same typed AST / logical Query IR**. It need not generate Haskell source code or implement general SQL semantics.

```text
Haskell DSL ------------+
                        +--> common AST --> semantic analysis --> Query IR
SQL-like parser --------+
```

The engine maps validated logical operations to physical algorithms based on available indexes, segments, and precomputed aggregates. Frontends must not depend on the on-disk storage layout.

### Query protocol: not yet selected

Query IR is a **separate protocol** from Arrow result streaming. Its initial representation, schema, and transport framing remain to be decided. A readable versioned format (e.g. JSON) may be useful for development and test fixtures, but this is **not yet a committed choice**.

Required properties:

- Explicit protocol/version identifiers.
- Deterministic semantics for time ranges, bucket boundaries, types, and ordering.
- Strict length and resource limits, validation, and actionable errors.
- Ability to save and replay a plan independently of the frontend.
- A path to compatibility testing as the frontend and engine evolve.

## 5. Results and Interoperability

**Decision: use Apache Arrow IPC Stream for tabular query results.**

- Rust emits a schema followed by successive Arrow record batches.
- Consumers may be written in Haskell, Go, Java, Kotlin, C, Rust, or another language with a compatible Arrow IPC reader.
- Result streams must support bounded-memory consumption and backpressure.
- Arrow is the **inter-process result format**, not a mandated on-disk storage format.
- Query control messages, execution status, and post-stream error reporting need explicit contracts beyond Arrow itself.

Ordinary Unix pipes provide a useful initial transport. They are byte streams, so framing is supplied by the selected protocol. For durable artifacts, a compatible Arrow file or stream representation can be chosen explicitly. The project should not label an arbitrary byte dump as an Arrow file without matching its format.

An illustrative pipeline (commands are **not implemented**):

```sh
timeframedb-query compile query.hs \
  | timeframedb-engine execute \
  | timeframedb-output table
```

Standard output should carry only the designated machine-readable stream; logs and diagnostics belong on standard error. Consumers and launchers should detect broken pipes, nonzero process exits, and incomplete results. A failed stream must not silently become a successful complete result.

For large query results, **streaming and bounded memory are requirements; end-to-end zero-copy is not**. Copies can still occur across kernel IPC and format decoding. Avoid unnecessary conversion or full-result materialization.

## 6. Storage Architecture: Research Hypotheses

The physical storage format has **not** been selected. The leading candidate is an append-oriented design with immutable segments, chronological metadata, sparse time indexes, and optional incremental aggregates.

```text
appendBatch(events)
      |
      v
validation / deduplication policy
      |
      v
immutable segment(s) + metadata
      |
      +----> sparse time index
      +----> optional aggregate summaries / timeframe hierarchy
      |
      v
atomic publication of a new readable version
      |
      v
range scan / aggregate / Arrow record batches
```

Candidates to study and benchmark:

1. **DuckDB-inspired reference:** columnar row groups, min/max zone maps, batched vectorized scans and aggregation.
2. **Append-oriented segments:** independently readable immutable files with compact time indexing.
3. **Aggregation structures:** fixed timeframe summaries, hierarchical aggregates, or an aggregate tree where justified.
4. **Encoding/compression:** raw fixed-width columns, delta-encoded timestamps, compression, and the cost of decoding.
5. **Read mechanisms:** buffered sequential I/O versus memory mapping, chosen by measurement.

An implementation inspired by DuckDB is **not equivalent to DuckDB**. The real database, on a pinned version and controlled configuration, remains a distinct baseline.

### Timeframe aggregation

For ordered, nonempty event ranges, OHLCV aggregates can be combined as:

```text
combine(A, B):
    open   = A.open
    high   = max(A.high, B.high)
    low    = min(A.low, B.low)
    close  = B.close
    volume = A.volume + B.volume
```

The implementation must define an empty-range identity, deterministic event ordering (including equal timestamps), numeric overflow behavior, and bucket boundaries. This composition is associative under those contracts, allowing aggregation over summaries instead of rescanning all original events.

**Hypothesis, not a performance claim:** indexed, precomputed timeframe aggregates may improve repeated aggregation queries. Fair benchmarks must compare them against **DuckDB with equivalently materialized summaries**, not only DuckDB rescanning raw ticks.

## 7. Consistent Append and Incremental Synchronization

Synchronization "by date" requires more than remembering the largest timestamp stored.

The design must distinguish:

- **Event time:** when an event actually occurred.
- **Source cursor / checkpoint:** confirmed progress in the source's ordering or pagination protocol.
- **Completeness watermark:** up to what event time the source can justify considering data complete.

Their relationship depends on source guarantees. In particular, `MAX(event_time)` must not automatically be interpreted as a complete synchronization boundary.

### Required properties

- Atomic visibility of a committed batch: readers never observe a partially committed batch.
- Durable, recoverable commits under an explicitly defined failure model.
- A defined strategy for retried/replayed batches and duplicate identity.
- Consistent snapshots for readers while a writer appends.
- Recovery tests for interruptions during data writing and metadata publication.
- Explicit handling or rejection of late/out-of-order events in the initial version.

A possible design is to write immutable segments, ensure their data is durable, and atomically publish a new manifest that identifies the visible segment set and source checkpoint. This remains a **design candidate**, not a verified protocol. Atomic replacement, directory synchronization, cross-platform filesystem semantics, cleanup of uncommitted files, and crash recovery must be specified and tested before durability is claimed.

One writer / multiple readers and monotonic append ordering are reasonable initial simplifications. Historical corrections, late data, concurrent writers, and distributed replication can be investigated later.

## 8. Resource Model and Operational Design

A major goal is a **small, predictable resource footprint**, particularly compared with the operational demands of a full PostgreSQL/TimescaleDB stack. The project should explicitly track index overhead, heap usage, page-cache pressure, startup time, and write amplification.

Separating Rust and Haskell into independent processes enables independent operating-system policies:

| Policy area | Rust engine | Haskell frontend / output tool |
| --- | --- | --- |
| CPU | Favor the execution engine under contention when justified | Lower weight for background work; avoid starving latency-critical query preparation |
| Memory | Measure and protect an appropriate working set | Constrain background or transient frontend memory as needed |
| I/O | Prioritize storage-critical I/O when appropriate | Separate policies for interactive versus background tools |
| Lifecycle | Can run as a service or single-shot CLI | Can be invoked on demand |

For Linux deployments, investigate systemd and cgroups v2 (`CPUWeight`, `MemoryLow`, `MemoryHigh`, `MemoryMax`, and applicable I/O controls). These are **deployment policies, not baked-in engine defaults**. CPU weight is relative, not reserved CPU time; memory protection is not a guaranteed pinned allocation. Separate processes do not automatically mean separate cgroups.

A key operational test is contention: benchmark latency and throughput when unrelated CPU, memory, and disk loads compete with the engine. Over-prioritizing Rust can be counterproductive if a starved query producer or output consumer blocks the end-to-end pipeline.

Real-time scheduling is not a default requirement.

## 9. Research and Benchmark Strategy

The initial investigation will compare algorithms before making strong architectural claims.

| Variant | Purpose |
| --- | --- |
| Real DuckDB | Production-grade external benchmark, with an explicit version and tuned, fair configuration. |
| C reference of DuckDB-inspired techniques | Isolate and understand selected columnar/zone-map/vectorized algorithms. |
| Rust reference of the same techniques | Control for implementation language and learn Rust storage-engine mechanics. |
| Specialized Rust append-oriented engine | Test the benefit and cost of narrower storage and indexing strategies. |
| DuckDB with precomputed aggregates | Fair baseline for repeated timeframe queries. |

### Benchmark families

- Batched ingestion at several batch sizes.
- Time-range scans for narrow and wide ranges.
- Raw-event aggregation into multiple timeframe sizes.
- Repeated reads of already computed aggregate data.
- One data source versus multiple symbols/partitions.
- Incremental sync, overlapping/replayed batches, and crash recovery.
- Concurrent append and snapshot reads.
- Resource contention and cold/warm cache conditions.

### Metrics

Throughput (events/s), latency (including p50/p95/p99 where relevant), peak RSS, heap/allocator usage, CPU utilization, bytes read/written, page faults, index bytes per raw-data byte, total stored bytes, write amplification, startup cost, and recovery time.

All implementations must use equivalent semantics and produce matching reference outputs. Avoid comparing unsafe/unflushed append with durable commits as if they had identical guarantees. Isolate in-memory algorithm experiments from on-disk database experiments, and separate first-build costs from query costs.

## 10. Non-goals for the First Version

- A general-purpose SQL database or full PostgreSQL compatibility.
- Per-row point-lookup optimization or a mandatory B-tree per column.
- Distributed coordination, sharding, consensus, or replication.
- Full support for arbitrary updates, deletes, joins, and user-defined functions.
- A bespoke cross-language columnar result protocol.
- Embedding GHC inside the Rust process, or requiring Rust–Haskell FFI.
- Demonstrating universal superiority over DuckDB.

## 11. Proposed Milestones

1. **Architecture audit:** study actual DuckDB storage, vectorization, aggregation, and durability mechanisms; pin sources and versions.
2. **Workload and correctness specification:** define event schemas, timeframe semantics, batching, source checkpoints, and required guarantees.
3. **Reference experiments:** implement minimal DuckDB-inspired algorithms in C and Rust and compare them with the real DuckDB engine.
4. **Rust storage prototype:** append, segment discovery, time-range scan, baseline aggregation, and correctness tests.
5. **Haskell DSL + Query IR:** build a typed frontend; drive the Rust engine through a versioned plan contract.
6. **Arrow IPC integration:** stream record batches to independent consumers, including a Haskell output CLI if useful.
7. **Consistency and recovery:** harden batch commit, idempotent sync, snapshot semantics, and crash testing.
8. **Specialized structures:** introduce sparse indexes, aggregate materialization, or compression individually, guided by benchmarks.
9. **Optional SQL-like parser and connectors:** keep them separate from the core engine.
10. **Research report:** publish reproducible measurements, negative findings, engineering trade-offs, and limitations.

## 12. Open Questions

These should remain open until specifications or experiments resolve them:

- What are the first supported event schemas and partitioning keys?
- What guarantees do target data sources provide for ordering, late events, and completeness?
- What is the correct durable commit protocol for target operating systems and filesystems?
- Which index layout offers the best time-range performance per byte of metadata?
- When do materialized timeframes justify their storage and ingestion overhead?
- How much memory can the engine consume under realistic workloads?
- What is the minimal versioned Query IR and plan transport?
- Which Arrow IPC libraries are practical for the initial Haskell client?
- How are errors, cancellation, and successful completion communicated alongside an Arrow result stream?
- How will project licensing, repository structure, API stability, and public naming/trademark availability be handled?

## 13. Scope of the Open Source Project

TimeframeDB is intended to be developed publicly, with reproducible experiments, readable implementations, documented constraints, and a clear separation between established design decisions and unverified hypotheses.

**The project is successful if it produces a correct, understandable, useful storage engine and credible research results—even if a mature existing database remains faster on some or most workloads.**

---

*This document records the project's initial direction. Technical choices not explicitly marked as decisions are hypotheses or proposals and should be revisited in light of evidence.*
