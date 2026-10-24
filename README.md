## Ford Haley

Computer Science · Systems & Distributed Infrastructure

### Professional Focus

I design and build distributed systems where correctness under partition and recovery time matter more than peak throughput. My work centers on consensus protocols, storage engines, and the operational properties that make them predictable: bounded memory, deterministic replay, and fail-fast behavior.

### Flagship Projects & Architecture

#### Raft-KV: A Log-Structured Key-Value Store with Raft Consensus

A distributed key-value store implementing the Raft consensus algorithm from scratch, with a log-structured merge-tree (LSM) storage engine.

**Architecture**
- Core components: a Raft consensus module (leader election, log replication, snapshotting), an LSM storage engine (memtable, sorted string tables, write-ahead log), and a thin gRPC API layer.
- Concurrency model: single-writer, multi-reader per node; the Raft loop runs in one goroutine, the storage engine uses a separate goroutine for compaction, and reads are served from a lock-free snapshot of the memtable.
- Storage layout: in-memory skiplist memtable (bounded to 64 MiB), immutable sorted string tables (SSTs) on disk with a block-based format (16 KiB blocks, CRC32C checksums), and a write-ahead log for durability before applying to the memtable.
- Wire protocol: gRPC with protobuf messages for AppendEntries, RequestVote, and client requests; each RPC carries a monotonic term and index.

**Trade-offs**
- Chose synchronous fsync on the write-ahead log over batching fsyncs for stronger durability, and paid a 2x write latency penalty under single-client workloads.
- Chose a skiplist memtable over a B-tree for simpler lock-free reads, and paid higher memory overhead (about 30% more per key) due to per-node pointers.
- Chose snapshotting at a fixed log size (1 GiB) over incremental log compaction for simpler recovery, and paid a longer snapshot install time (up to 2 seconds) on follower catch-up.

**Results**
- 12,000 writes/sec at p50 0.8 ms, p95 1.4 ms, p99 2.1 ms latency on a 3-node cluster, 1 KiB payloads, with synchronous fsync, measured on commodity VMs (4 vCPU, 8 GiB RAM, SSD).
- 40,000 reads/sec at p50 0.3 ms, p95 0.6 ms, p99 0.9 ms from a single node, serving reads from the memtable snapshot, same hardware.
- Recovery time after leader crash: 1.8 seconds to elect a new leader and apply the last 10,000 log entries, measured with a 3-node cluster and 1 GiB log.
- Write amplification during compaction: 8x on average with a 64 MiB memtable and 4:1 size ratio between levels, measured over a 10 GiB dataset.

#### Stream-Proc: A Deterministic Stream Processing Engine

A single-node stream processing engine with deterministic replay and backpressure control, designed for exactly-once semantics on a single partition.

**Architecture**
- Core components: a source connector (reads from Kafka), a processing pipeline (map, filter, windowed aggregations), a sink connector (writes to Kafka), and a state store backed by RocksDB.
- Concurrency model: a single-threaded event loop processes events in order; state mutations are serialized through the loop, and checkpointing runs in a separate thread that snapshots the state store.
- Storage layout: RocksDB for state (key-value), with periodic snapshots to a local filesystem and a write-ahead log for state changes between snapshots.
- Wire protocol: Kafka protocol for input and output; internal state is serialized as protobuf.

**Trade-offs**
- Chose single-threaded processing over multi-threaded for deterministic replay and simpler exactly-once semantics, and paid a throughput ceiling of about 50k events/sec on a single core.
- Chose synchronous checkpointing (stop-the-world) over asynchronous to guarantee a consistent snapshot, and paid a 100 ms pause every 10 seconds.
- Chose RocksDB over an in-memory store for state durability, and paid 20% higher latency on state access due to disk I/O.

**Results**
- 45,000 events/sec sustained throughput with 1 KiB payloads, single-threaded, on a 4 vCPU, 8 GiB VM, measured over 10 minutes.
- p50 2 ms, p95 4 ms, p99 6 ms end-to-end latency from source to sink, under 10k events/sec load, same hardware.
- Deterministic replay: reprocessing the same 1 million events from a checkpoint produced identical output state, verified by hash comparison.
- Backpressure: when the sink is slow, the pipeline applies backpressure to the source, reducing input rate by 80% within 200 ms, preventing unbounded memory growth.

### Technical Foundation

**Core Systems**
- `Go` (primary language for both projects)
- `gRPC` and `protobuf` for wire protocols
- `Raft` consensus algorithm (implemented from the paper)
- `RocksDB` for state storage

**Storage & Data**
- LSM tree design, SSTable format, write-ahead logging
- `Kafka` for stream ingestion and output
- `BoltDB` (used in experiments for comparison)

**Infrastructure & Observability**
- `Docker` and `docker-compose` for local clusters
- `Prometheus` and `Grafana` for metrics and dashboards
- `go test -race` and `go vet` for correctness checks

### How I Build

- I write tests that verify invariants under failure injection, because a system that passes happy-path tests is not proven correct.
- I profile before optimizing, and I keep a record of the bottleneck and the fix, because guesses waste more time than measurements.
- I keep all queues bounded and apply backpressure at the source, because unbounded memory is a latency bomb.
- I design for deterministic replay, because the ability to reproduce a failure is the first step to fixing it.

### Current Explorations

- The Raft paper (Ongaro and Ousterhout, 2014) — studying the leader election and log replication edge cases to harden my implementation against split-brain scenarios.
- The LSM-tree paper (O'Neil et al., 1996) — applying its compaction and merge policies to reduce write amplification in my storage engine.
- The Linux kernel's `io_uring` subsystem — evaluating whether an async I/O model can improve my storage engine's throughput without sacrificing durability.

### Contact

GitHub: [@lnvYaqdnRkXhWM](https://github.com/lnvYaqdnRkXhWM)