# Backend & Data Platform Engineering — Master Curriculum

## 1. Linux, Operating Systems, and Runtime Foundations

### 1.1 Processes, Threads, and Scheduling
- User space vs kernel space
- System call execution and context switching overhead
- Process lifecycle (`fork`, `exec`, `wait`, `exit`, zombie and orphan processes)
- Kernel threads vs user-space threads vs coroutines
- OS scheduler internals (CFS, time slicing, priorities, run queues)
- CPU affinity, NUMA topology, and cache-line locality
- Signal handling (`SIGTERM`, `SIGINT`, `SIGKILL`, `SIGHUP`) and signal safety

### 1.2 Memory Management and Storage I/O
- Virtual memory, page tables, and TLB
- Page faults (minor vs major) and copy-on-write
- Kernel page cache, dirty pages, and `fsync` / `fdatasync` durability guarantees
- Memory-mapped files (`mmap`) and zero-copy I/O (`sendfile`, `splice`)
- Buffered I/O vs direct I/O (`O_DIRECT`)
- OOM killer scoring and cgroup memory limits
- Memory fragmentation and allocator architectures (`ptmalloc`, `jemalloc`, `tcmalloc`)

### 1.3 I/O Models and Multiplexing
- Blocking vs non-blocking I/O
- Synchronous vs asynchronous I/O
- `select` and `poll` limitations
- `epoll` internals (edge-triggered vs level-triggered, interest list, ready list)
- `io_uring` submission and completion ring buffers
- File descriptors, descriptor limits (`ulimit`), and socket buffers

### 1.4 Concurrency Primitives and Synchronization
- Race conditions, critical sections, and memory visibility
- Mutexes, spinlocks, semaphores, and read-write locks
- Condition variables and barriers
- Lock-free programming, Compare-And-Swap (CAS), and memory ordering
- Deadlocks, livelocks, starvation, and priority inversion
- Thread pools, work stealing, and queue sizing

### 1.5 Linux Diagnostics and Isolation Primitives
- Linux namespaces (PID, NET, MNT, UTS, IPC, USER)
- Control groups (cgroups v1 vs v2: CPU throttling, memory, I/O limits)
- Overlay filesystems (`OverlayFS`) and union mounts
- System inspection tools (`strace`, `lsof`, `ss`, `vmstat`, `iostat`, `perf`, `sar`)
- `/proc` and `/sys` pseudo-filesystems

---

## 2. Networking, Transport Protocols, and HTTP

### 2.1 Network Stack and Addressing
- OSI model vs TCP/IP stack
- IPv4 vs IPv6, CIDR notation, and subnetting
- ARP, routing tables, MTU, and packet fragmentation
- DNS resolution lifecycle, recursive vs authoritative servers, TTL, and negative caching
- DNS record types (`A`, `AAAA`, `CNAME`, `SRV`, `TXT`) and DNS-based traffic routing
- NAT, SNAT, DNAT, and ephemeral port exhaustion
- Sockets API (`socket`, `bind`, `listen`, `accept`, `connect`) and listen backlog (`SOMAXCONN`, SYN queue, accept queue)

### 2.2 Transport Layer Internals
- TCP segment structure and sequence/acknowledgment numbers
- TCP 3-way handshake and 4-way teardown (`TIME_WAIT`, `CLOSE_WAIT` states)
- TCP flow control and sliding window mechanics
- TCP congestion control (slow start, congestion avoidance, AIMD, CUBIC, BBR)
- Head-of-line blocking at the transport layer
- TCP keep-alive vs application-level heartbeats
- Nagle's algorithm and `TCP_NODELAY`
- UDP semantics, packet loss trade-offs, and multicast
- QUIC protocol internals (0-RTT/1-RTT handshakes, stream multiplexing, connection migration)

### 2.3 HTTP Protocol Evolution and Semantics
- Client-server architecture and request-response lifecycle
- HTTP message framing (start-line, headers, empty line, body)
- HTTP/1.1 persistent connections (`Keep-Alive`) and pipelining limits
- HTTP/2 binary framing, HPACK header compression, stream multiplexing, and TCP HOL blocking
- HTTP/3 over QUIC and QPACK compression
- HTTP methods (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`) and safety/idempotency semantics
- HTTP status code families (1xx–5xx) and semantic error mapping
- Content negotiation, transfer encoding (`chunked`), and compression (`gzip`, `brotli`, `zstd`)

### 2.4 HTTP Headers and State Management
- Core request and response headers (`Host`, `Content-Type`, `Content-Length`, `Accept`, `User-Agent`, `Connection`)
- Caching headers (`Cache-Control`, `ETag`, `If-None-Match`, `Last-Modified`, `Vary`, `Age`)
- Cross-origin mechanics (`Origin`, `Referer`, CORS preflight requests, `Access-Control-*`)
- Stateless vs stateful server architectures
- HTTP cookies (`HttpOnly`, `Secure`, `SameSite`, `Domain`, `Path`)
- Server-side sessions, distributed session stores, expiration, and session fixation prevention
- Session affinity (sticky sessions) vs shared externalized state

### 2.5 TLS, Cryptography in Transit, and Edge Proxying
- SSL vs TLS evolution
- Symmetric vs asymmetric cryptography and key exchange (ECDHE, forward secrecy)
- X.509 certificates, Certificate Authorities, trust chains, and SANs
- TLS 1.2 vs TLS 1.3 handshake mechanics and 0-RTT resumption risks
- Certificate validation, OCSP stapling, and automated certificate rotation
- Mutual TLS (mTLS) for service-to-service identity
- Forward proxies vs reverse proxies
- L4 (TCP/UDP passthrough, DSR) vs L7 (HTTP-aware routing, TLS termination) load balancing

### 2.6 RPC and Real-Time Communication Protocols
- Protocol Buffers wire format and schema evolution rules
- gRPC over HTTP/2 (unary, server-streaming, client-streaming, bidirectional streaming)
- gRPC deadlines, cancellation propagation, and interceptors
- Long polling vs Server-Sent Events (SSE)
- WebSocket handshake (`Upgrade`), frame structure, heartbeats, and stateful connection scaling
- Backpressure and fan-out architectures for real-time backends

---

## 3. Python Engineering and Runtime Internals

### 3.1 CPython Memory Model and Object Internals
- `PyObject` structure, reference counting, and cyclic garbage collector (generational GC)
- Object allocation (`pymalloc`, arenas, pools, blocks)
- Mutable vs immutable types, interning, and copy vs deepcopy semantics
- `__slots__` memory layout and descriptor protocol (`__get__`, `__set__`, `__delete__`)
- Weak references (`weakref`) and memory leak patterns in long-running services

### 3.2 Execution Model, GIL, and Concurrency
- Bytecode compilation, `.pyc` files, and the CPython evaluation loop
- Global Interpreter Lock (GIL) mechanics and I/O release boundaries
- Free-threaded Python (`PEP 703`, no-GIL) implications and synchronization
- I/O-bound vs CPU-bound workload classification
- `threading` module, thread safety, and lock contention
- `multiprocessing` (`fork` vs `spawn` vs `forkserver`), IPC pipes/queues, and shared memory (`multiprocessing.shared_memory`)
- `concurrent.futures` (`ThreadPoolExecutor`, `ProcessPoolExecutor`)

### 3.3 Asynchronous Python (`asyncio`) Internals
- Generators, `yield from`, coroutines, and `async`/`await` state machines
- Event loop architecture (selector/epoll integration, task queue, timers)
- Tasks, Futures, `asyncio.gather`, `asyncio.TaskGroup`, and structured concurrency
- Cancellation propagation, `CancelledError` handling, and shielding
- Async synchronization primitives (`asyncio.Lock`, `Semaphore`, `Queue`, `Event`)
- Blocking the event loop diagnostics and `asyncio.to_thread` offloading
- `uvloop` and Cythonized event loops

### 3.4 Type System, Interfaces, and Metaprogramming
- Static typing (`typing`, `Protocol`, `TypeVar`, `ParamSpec`, `Generic`, `Annotated`)
- Structural subtyping vs nominal subtyping
- Dataclasses, `TypedDict`, Pydantic validation internals, and runtime overhead
- Context managers (`__enter__`/`__exit__`, `contextlib`, `AsyncExitStack`)
- Decorators, closures, `functools.wraps`, and `contextvars` for request-scoped context
- Abstract Base Classes (`abc`) and dependency injection patterns

### 3.5 Production Python Packaging, Tooling, and Profiling
- Virtual environments, wheels, editable installs, and `pyproject.toml` (`PEP 517`/`518`/`621`)
- Reproducible lockfiles (`uv`, `pip-tools`, `poetry`)
- Static analysis and type checking (`mypy`, `pyright`, `ruff`)
- CPU profiling (`cProfile`, `py-spy`, `scalene`, flamegraphs)
- Memory profiling (`tracemalloc`, `memray`, heap dump analysis)
- WSGI vs ASGI specifications (`uvicorn`, `gunicorn`, worker process management)

---

## 4. Backend Service Architecture and API Engineering

### 4.1 Layered Service Architecture and Request Lifecycle
- Web server vs application server vs framework responsibilities
- Routing mechanics (radix trees / trie routers, path parameters, query parsing, route precedence)
- Middleware pipeline execution order (onion/chain-of-responsibility pattern)
- Request context propagation (`request_id`, `trace_id`, deadlines, auth principal)
- Controllers/handlers, service layer, repository/DAO layer, and domain models
- Data Transfer Objects (DTOs) vs persistence models vs domain entities
- Dependency inversion and clean/hexagonal architecture boundaries

### 4.2 Serialization, Validation, and Transformation
- Text vs binary serialization (JSON, MessagePack, Avro, Protobuf)
- Streaming parsers vs DOM parsers and memory amplification risks
- Schema validation (syntactic vs semantic validation, fail-fast vs error aggregation)
- Input sanitization, canonicalization, and type coercion pitfalls
- Pagination models (offset/limit vs cursor/keyset pagination, consistency under concurrent writes)
- Filtering, sorting, sparse fieldsets, and response envelopes

### 4.3 REST API Design and Contracts
- Resource modeling, URI hierarchy, and sub-resource semantics
- Versioning strategies (URI, header, media-type negotiation) and deprecation lifecycles
- Idempotency keys, storage, concurrency locking, and replay semantics
- Optimistic concurrency control with `ETag` and `If-Match`
- Long-running async operations (`202 Accepted`, status polling endpoints, callback URLs)
- OpenAPI / JSON Schema specifications (contract-first vs code-first, codegen, breaking-change detection)
- Rate-limit response headers (`Retry-After`, `X-RateLimit-*`) and RFC 9457 Problem Details errors

### 4.4 Webhooks and Outbound Event Delivery
- Webhook registration, subscription filtering, and verification handshakes
- Payload signing with HMAC-SHA256, timestamp headers, and replay-window validation
- Delivery workers, exponential backoff with jitter, and dead-letter handling
- Circuit breaking misbehaving subscriber endpoints
- Receiver-side deduplication, asynchronous ingestion, and fast acknowledgment (`2xx` before processing)

### 4.5 Error Handling, Configuration, and Service Lifecycle
- Domain errors vs infrastructure errors vs transient faults
- Global exception handlers and error boundary isolation
- Twelve-Factor App principles (stateless processes, port binding, disposability, dev/prod parity)
- Production configuration management (env vars, typed config schemas, secret injection, dynamic config/feature flags)
- Graceful shutdown lifecycle (`SIGTERM` handling, readiness probe flipping, connection draining, in-flight request completion, resource cleanup)

---

## 5. Caching, Full-Text Search, and Asynchronous Job Systems

### 5.1 Caching Architectures and Strategies
- Memory hierarchy and latency budgets (L1/L2 CPU, RAM, NVMe, network)
- Client-side, CDN/edge, reverse-proxy, in-process, and distributed caches
- Read strategies (cache-aside/lazy loading, read-through)
- Write strategies (write-through, write-behind/write-back, write-around)
- Cache eviction policies (LRU, LFU, TinyLFU/W-TinyLFU, FIFO, random)
- Cache expiration (TTL, jittered TTL, sliding expiration)
- Cache stampede / thundering herd mitigation (request coalescing/singleflight, probabilistic early expiration, mutex locks)
- Cache penetration (Bloom filters, null-object caching) and hot-key mitigation (local L1 cache, key salting)
- Cache consistency and invalidation patterns (versioned keys, CDC-driven invalidation)

### 5.2 Redis Internals and Advanced Usage
- Single-threaded event loop and I/O multiplexing model
- Internal data structures (SDS, ziplist/listpack, skiplist, hash tables, hyperloglog, bitmaps, streams)
- Persistence mechanisms (RDB snapshots via `fork` COW, AOF rewrite, hybrid RDB+AOF)
- Redis pipelining, Lua scripting, and atomic execution semantics
- Redis Replication, Sentinel failover, and Redis Cluster (16,384 hash slots, `MOVED`/`ASK` redirections)
- Memory limits, `maxmemory-policy`, and big-key latency spikes

### 5.3 Task Queues and Background Job Processing
- Broker-backed task queues vs polling database queues (`SELECT ... FOR UPDATE SKIP LOCKED`)
- Producer, broker, worker, and result-backend topology
- Delivery guarantees (at-most-once, at-least-once, effective-once via idempotency)
- Visibility timeouts, heartbeats, ack/nack semantics, and redelivery
- Retry policies, exponential backoff, jitter, and Dead-Letter Queues (DLQ)
- Priority queues, fair scheduling across tenants (weighted fair queuing), and noisy-neighbor isolation
- Cron/delayed/scheduled job engines (timing wheels, sorted-set pollers)

### 5.4 Full-Text Search and Elasticsearch Internals
- Forward index vs inverted index structure
- Text analysis pipeline (character filters, tokenizers, token filters, stemming, n-grams)
- Relevance scoring (TF-IDF, BM25, field boosts, vector/hybrid k-NN search)
- Lucene segments, immutability, refresh interval, translog, and segment merging
- Distributed indexing, shard routing, primary/replica sync, and scatter-gather search
- Deep pagination (`search_after` vs `from/size`), index aliases, and zero-downtime reindexing

---

## 6. Storage Engine Internals and Database Foundations

### 6.1 Data Models and On-Disk Layouts
- Row-oriented vs column-oriented storage layouts
- Slotted pages, heap files, tuples, and alignment/padding
- Variable-length data, overflow pages, and TOAST-style out-of-line storage
- Append-only logs and log compaction

### 6.2 Indexing Structures and Trade-offs
- B-Tree and B+Tree internals (node fanout, page splits, merges, fill factor, range scans)
- LSM-Tree internals (memtable, WAL, immutable memtable, SSTables, sparse indexes, fence pointers)
- LSM compaction strategies (size-tiered, leveled, FIFO) and write/read/space amplification trade-offs
- Hash indexes (static vs extendible vs linear hashing)
- Specialized indexes (bitmap, inverted, R-Tree/GiST, GIN, BRIN)
- Covering indexes (index-only scans) and composite index column ordering

### 6.3 Transactions, Concurrency Control, and Recovery
- ACID properties (atomicity, consistency, isolation, durability)
- Write-Ahead Logging (WAL), ARIES recovery (analysis, redo, undo), checkpoints, and steal/no-force buffer policies
- Pessimistic concurrency control (2PL, shared/exclusive locks, intention locks, deadlock detection via wait-for graphs)
- Optimistic concurrency control (OCC) and timestamp ordering
- Multi-Version Concurrency Control (MVCC) and snapshot visibility
- ANSI SQL isolation levels (Read Uncommitted, Read Committed, Repeatable Read, Serializable)
- Isolation anomalies (dirty reads, non-repeatable reads, phantom reads, lost updates, write skew, read-only transaction anomaly)
- Serializable Snapshot Isolation (SSI) internals (SIREAD locks, rw-antidependency tracking)

---

## 7. SQL and PostgreSQL Internals

### 7.1 Advanced SQL and Query Patterns
- Window functions (`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`/`LEAD`, frame clauses `ROWS` vs `RANGE`)
- Common Table Expressions (CTEs), materialization fences, and recursive CTEs
- Lateral joins, `UPSERT` (`INSERT ... ON CONFLICT`), and `MERGE`
- Grouping sets, `ROLLUP`, `CUBE`, and conditional aggregation (`FILTER`)
- Null semantics (`IS DISTINCT FROM`, three-valued logic) and anti-join pitfalls (`NOT IN` vs `NOT EXISTS`)

### 7.2 PostgreSQL Architecture and Storage Internals
- Process-per-connection architecture, Postmaster, and background workers
- Shared memory layout (`shared_buffers`, `wal_buffers`, `CLOG`/`pg_xact`, lock tables)
- Heap tuple header internals (`xmin`, `xmax`, `ctid`, infomasks, visibility maps)
- MVCC update/delete mechanics and HOT (Heap-Only Tuple) updates
- `VACUUM` vs `AUTOVACUUM`, transaction ID (`xid`) wraparound, freezing, and bloat remediation
- TOAST storage strategies (`PLAIN`, `EXTENDED`, `EXTERNAL`, `MAIN`)

### 7.3 Query Execution, Planner, and Indexing in Postgres
- Query lifecycle (parser, analyzer, rewriter, planner/optimizer, executor)
- Cost-based optimizer (table statistics, `pg_statistic`, histograms, MCV lists, correlation, selectivity estimation)
- Scan nodes (Sequential, Index, Index-Only, Bitmap Heap/Index Scan, TID Scan)
- Join algorithms (Nested Loop, Hash Join, Merge Join) and memory (`work_mem`) spill behavior
- PostgreSQL index types (`B-Tree`, `Hash`, `GIN`, `GiST`, `SP-GiST`, `BRIN`), partial indexes, and expression indexes
- `EXPLAIN (ANALYZE, BUFFERS, VERBOSE, WAL)` interpretation and plan regression diagnosis

### 7.4 Locking, High Availability, and Scaling Postgres
- Relation-level locks (`ACCESS SHARE` through `ACCESS EXCLUSIVE`) and safe zero-downtime DDL migrations
- Row-level locks (`FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE`, `SKIP LOCKED`, `NOWAIT`)
- Connection pooling architectures (session vs transaction vs statement pooling, `PgBouncer` internals, prepared statement caveats)
- Streaming replication (physical WAL shipping, replication slots, `synchronous_commit` modes)
- Logical replication (publications, subscriptions, `pgoutput` decoding, DDL limitations)
- Declarative partitioning (range, list, hash), partition pruning, and shard-key selection

---

## 8. Distributed Systems Theory, Consensus, and Replication

### 8.1 Foundational Limits, Time, and Ordering
- Asynchronous vs synchronous vs partially synchronous network models
- FLP impossibility theorem and CAP / PACELC trade-offs
- Failure modes (crash-stop, crash-recovery, omission, network partitions, Byzantine faults)
- Split-brain scenarios and network asymmetry
- Physical clocks, NTP drift, clock skew, and monotonicity issues
- Logical time (Lamport timestamps, vector clocks, version vectors, dotted version vectors)
- Hybrid Logical Clocks (HLC) and bounded uncertainty clocks (TrueTime)

### 8.2 Consistency Models and Anomalies
- Data-centric consistency (strict/linearizability, sequential, causal, PRAM/FIFO, eventual)
- Client-centric consistency (read-your-writes, monotonic reads, monotonic writes, writes-follow-reads)
- Linearizability vs serializability vs strict serializability
- Conflict resolution strategies (Last-Write-Wins pitfalls, application-level merge, CRDTs: state-based CvRDT vs operation-based CmRDT)

### 8.3 Replication Architectures and Quorums
- Single-leader replication (synchronous vs asynchronous, failover promotion, replication lag, split-brain fencing)
- Multi-leader replication (write conflicts, conflict detection, active-active multi-region trade-offs)
- Leaderless / Dynamo-style replication (`N`, `W`, `R` quorum intersection)
- Sloppy quorums and hinted handoff
- Read repair and background anti-entropy via Merkle trees

### 8.4 Consensus, Membership, and Coordination
- Paxos fundamentals (Basic Paxos, Multi-Paxos, Fast Paxos, acceptor/proposer quorums)
- Raft internals (leader election, randomized election timeouts, log replication, safety proof, log compaction/snapshotting, membership changes)
- ZooKeeper Atomic Broadcast (ZAB) and Viewstamped Replication
- Membership and failure detection (Phi Accrual Failure Detector, SWIM protocol, Gossip dissemination)
- Distributed locks, leases, and fencing tokens (monotonically increasing epochs)
- Coordination services (`etcd`, ZooKeeper, Chubby patterns: watches, ephemeral nodes, leader election)

### 8.5 Distributed Transactions and Reliability Patterns
- Two-Phase Commit (2PC) coordinator failure blocking and Three-Phase Commit (3PC)
- Calvin / deterministic execution and Percolator-style distributed transactions
- Saga pattern (choreography vs orchestration, compensating transactions, semantic locks)
- Try-Confirm-Cancel (TCC) pattern
- Transactional Outbox and Inbox/Idempotent Receiver patterns (polling vs CDC-based relay)

---

## 9. Distributed Databases and Object Storage

### 9.1 Partitioning, Routing, and Distributed NoSQL Internals
- Range partitioning vs hash partitioning vs consistent hashing (virtual nodes, bounded loads)
- Secondary index partitioning (local/document-partitioned vs global/term-partitioned)
- Partition rebalancing, hot partitions, and scatter-gather queries
- Wide-column store internals (Cassandra/ScyllaDB: partition keys, clustering keys, tombstones, compaction, gossip ring)
- Key-value / document store internals (DynamoDB: request routers, storage nodes, Paxos replication, adaptive capacity)

### 9.2 NewSQL and Distributed SQL Architectures
- Spanner-style architecture (TrueTime, Paxos groups per range, 2PL + 2PC across shards, lock-free read-only snapshots)
- CockroachDB / TiDB / YugabyteDB architecture (Raft range groups, KV-to-SQL mapping, HLC read restarts, distributed MVCC)
- Distributed query execution (predicate pushdown, distributed joins, broadcast vs shuffle joins)
- Multi-region topologies (geo-partitioning, follower reads, survival goals)

### 9.3 Object Storage Internals
- Flat namespace, key-prefix hashing, and metadata vs blob storage separation
- Multipart uploads, resumable transfers, and ETag/checksum verification
- Strong read-after-write consistency models in modern object stores
- Presigned URLs, access control, and direct-to-storage client offload patterns
- Storage classes, lifecycle policies, versioning, object lock (WORM), and small-file compaction trade-offs
- Throughput scaling (prefix parallelization, byte-range parallel reads, request-rate throttling)

---

## 10. Event Streaming, CDC, and Apache Kafka Internals

### 10.1 Log-Based Event Architecture
- Event-carried state transfer vs event notification vs event sourcing
- Append-only distributed log abstractions
- Queues vs pub/sub logs (message deletion vs offset retention)
- Schema design (Avro, Protobuf, JSON Schema) and compatibility modes (`BACKWARD`, `FORWARD`, `FULL`, transitive variants)
- Schema Registry architecture and wire-format magic-byte framing

### 10.2 Apache Kafka Broker and Storage Internals
- Topics, partitions, segments, `.log`, `.index`, and `.timeindex` files
- Zero-copy message delivery (`sendfile`) and OS page cache reliance
- Controller architecture (ZooKeeper legacy vs KRaft Raft-based metadata quorum)
- Replication internals (Leader, Followers, High Watermark, Leader Epoch, In-Sync Replicas `ISR`)
- Unclean leader election, `min.insync.replicas`, and durability vs availability trade-offs
- Log retention (time/size-based deletion) vs log compaction (tombstones, cleaner threads)

### 10.3 Kafka Producers and Consumers
- Producer internals (`RecordAccumulator`, batching `linger.ms`/`batch.size`, compression, custom partitioners)
- Producer reliability (`acks=0/1/all`, `enable.idempotence`, PID/sequence numbers, `max.in.flight.requests.per.connection`)
- Transactional producers and Exactly-Once Semantics (EOS: transaction coordinator, `_transaction_state`, `read_committed` isolation)
- Consumer groups, offset management (`__consumer_offsets`), and `auto.offset.reset`
- Rebalance protocols (Eager vs Cooperative Sticky vs KIP-848 next-gen consumer rebalance)
- Consumer liveness (`heartbeat.interval.ms`, `session.timeout.ms`, `max.poll.interval.ms`) and rebalance storms
- Lag monitoring, backpressure, dead-letter topics, and poison-pill handling

### 10.4 Change Data Capture (CDC) Internals
- Polling-based vs trigger-based vs log-based CDC
- Debezium internals (PostgreSQL WAL logical decoding `pgoutput`, MySQL binlog, MongoDB oplog)
- Initial consistent snapshotting vs streaming phase handoff (DBLog watermark algorithm)
- Replication slot retention risks (disk exhaustion on idle slots, heartbeat tables)
- Tombstone events, schema change events, and downstream out-of-order / idempotent merge handling

---

## 11. Stream Processing and Real-Time Compute Engines

### 11.1 Time Semantics, Windows, and Watermarks
- Event time vs ingestion time vs processing time
- Watermarks (heuristic vs punctuated, multi-partition watermark alignment, idle source handling)
- Late-arriving data handling (allowed lateness, side outputs, retractions/updates)
- Window types (tumbling, sliding/hopping, session, global/count windows)
- Triggers and evictors

### 11.2 Stateful Stream Processing and Fault Tolerance
- Keyed state vs operator state (ValueState, ListState, MapState, ReducingState)
- State backends (heap vs out-of-core RocksDB, incremental checkpointing, write amplification)
- State TTL and state schema evolution
- Chandy-Lamport distributed snapshotting (checkpoint barriers, aligned vs unaligned checkpoints, savepoints)
- Stream-table duality (KStream/KTable, changelog streams, temporal joins, interval joins)
- End-to-end exactly-once sinks (idempotent upserts vs two-phase commit sinks)

### 11.3 Stream Processing Engine Architectures
- Micro-batch (Spark Structured Streaming) vs continuous operator (Apache Flink) execution models
- Flink runtime (JobManager, TaskManager, task slots, operator chaining, network buffer pools, credit-based flow control)
- Backpressure propagation and detection in streaming DAGs
- Skewed stream keys (two-phase local-global aggregation, key salting)
- Kafka Streams topology, embedded RocksDB state stores, and standby replicas

---

## 12. Distributed Batch Processing, Apache Spark, Lakehouse, and OLAP Systems

### 12.1 Apache Spark Execution Internals
- Driver, Executors, Cluster Manager, and job/stage/task hierarchy
- RDD lineage, narrow vs wide transformations, and stage boundaries
- Catalyst Optimizer (unresolved logical plan, catalog analysis, logical optimization, physical planning, cost-based optimization)
- Project Tungsten (off-heap binary memory management `UnsafeRow`, cache-aware computation, whole-stage code generation)
- Adaptive Query Execution (AQE: dynamic partition coalescing, dynamic join strategy switching, skew join optimization)

### 12.2 Spark Memory, Shuffle, and Performance Tuning
- Unified memory management (execution vs storage memory, spill thresholds, off-heap memory, Python worker memory in PySpark/Arrow)
- Shuffle internals (SortShuffleManager, bypass merge-sort, index/data files, external shuffle service, push-based shuffle)
- Join execution strategies (Broadcast Hash Join, Shuffle Hash Join, Sort-Merge Join)
- Data skew symptoms, straggler tasks, salting techniques, and isolated key processing
- Partitioning vs bucketing, small-file explosion, and `coalesce` vs `repartition`
- Speculative execution, dynamic allocation, and OOM failure diagnosis (Driver vs Executor vs Python worker)

### 12.3 Columnar File Formats and Lakehouse Table Formats
- Apache Parquet and ORC internals (row groups, column chunks, pages, dictionary/RLE/bit-packing/delta encodings, min/max stats, bloom filters)
- Data lake failure modes (non-atomic directory overwrites, listing overhead, schema drift, partial reads)
- Apache Iceberg internals (catalog pointer, metadata files, manifest lists, manifest files, snapshot isolation, hidden partitioning, partition evolution)
- Delta Lake internals (`_delta_log` JSON commits, checkpoints, optimistic concurrency, deletion vectors, Liquid Clustering)
- Apache Hudi internals (Copy-on-Write vs Merge-on-Read, timeline, indexing)
- Lakehouse maintenance (compaction/bin-packing, Z-ordering/clustering, snapshot expiration, orphan file cleanup, time travel)

### 12.4 Distributed Analytical and OLAP Engines
- MPP query engines (Trino / Presto: coordinator, workers, split scheduling, dynamic filtering, spill-to-disk)
- Embedded vectorized OLAP (DuckDB: columnar vectorized execution, morsel-driven parallelism, zero-copy Arrow integration)
- Real-time OLAP stores (ClickHouse: MergeTree engine family, primary sparse indexes, parts/merges, materialized views; Apache Pinot: segments, star-tree indexes, upsert/realtime ingestion)
- Query federation vs materialization trade-offs

---

## 13. Data Engineering Algorithms, System Primitives, and High-Scale Architecture Patterns

### 13.1 Core Backend and Data-System Primitives
- Consistent hashing with virtual nodes and bounded-load variants
- Rendezvous (Highest Random Weight) hashing
- Probabilistic data structures (Bloom filters, Counting Bloom, Cuckoo filters, HyperLogLog, Count-Min Sketch, t-digest)
- Distributed unique ID generation (UUIDv4 vs UUIDv7, ULID, Snowflake ID, Twitter/Instagram shard-ID layouts, clock-regression handling)
- Rate-limiting algorithms (Token Bucket, Leaky Bucket, Fixed Window, Sliding Window Log, Sliding Window Counter, distributed Redis/Lua atomic limiters)
- Geospatial indexing and proximity search (Geohash, QuadTree, Google S2, Uber H3 hexagonal hierarchical indexing, R-Trees)

### 13.2 Concurrency, Scheduling, and Low-Level Design Patterns
- Thread-safe bounded blocking queues, ring buffers (LMAX Disruptor patterns), and connection pools
- LRU / LFU / TTL concurrent caches (striped locking, doubly-linked list + hash map)
- In-memory pub/sub brokers and delayed task schedulers
- State machines and idempotent workflow transitions
- DAG dependency resolution, topological scheduling, and cycle detection

### 13.3 Uber-Grade High-Scale System Design Patterns
- Real-time geospatial matching and dispatch systems (high-frequency location ingestion, spatial indexing, surge/ETA computation, stateful matching)
- High-throughput financial ledger and payment/billing platforms (double-entry bookkeeping, idempotent state transitions, reconciliation pipelines, exactness guarantees)
- Large-scale telemetry, metrics, and log ingestion pipelines (edge buffering, partitioned streaming, tiered cold/hot storage, downsampling)
- Real-time ML feature stores and streaming-batch parity (online low-latency store + offline lakehouse backfill, point-in-time joins)
- Multi-tenant distributed job scheduling and query-routing platforms (resource quotas, fair-share preemption, admission control)
- Global multi-region active-active and active-passive service architectures (cell-based architecture, data locality, failover routing)

---

## 14. Security, Identity, and Data Governance

### 14.1 Authentication and Authorization
- Password hashing (Argon2id, bcrypt, scrypt, salting, peppering)
- Session tokens vs bearer tokens vs opaque reference tokens
- JWT internals (header, payload, signature, `alg=none` and key-confusion attacks, JWKS rotation, short TTL + refresh token rotation)
- OAuth 2.0 flows (Authorization Code + PKCE, Client Credentials, token introspection/revocation)
- OpenID Connect (OIDC) ID tokens and federated identity
- Authorization models (RBAC, ABAC, ReBAC / Zanzibar-style tuple graphs)
- Row-level and column-level access control in data platforms

### 14.2 Application, Network, and Infrastructure Security
- OWASP API Top 10 (BOLA/IDOR, BFLA, SSRF, SQL injection, mass assignment, resource exhaustion)
- Secrets management (HashiCorp Vault, cloud Secrets Manager, dynamic short-lived credentials, secret rotation)
- Key Management Service (KMS) internals and envelope encryption (DEK/KEK)
- Encryption at rest vs in transit vs client-side field-level encryption
- Network segmentation (VPCs, private subnets, security groups, NACLs, PrivateLink, service meshes)
- Supply-chain security (SBOMs, container image signing, vulnerability scanning, least-privilege IAM)

### 14.3 Data Governance, Quality, and Lineage
- Metadata catalogs (Hive Metastore, AWS Glue, Iceberg REST Catalog, Unity/Polaris)
- Column-level and dataset-level data lineage tracking
- Data quality enforcement (schema contracts, freshness/completeness/distribution assertions, circuit-breaking bad partitions)
- PII classification, tokenization, masking, anonymization, and GDPR/CCPA right-to-erasure in immutable lakes/logs

---

## 15. Cloud Infrastructure, Containers, Kubernetes, and Platform Orchestration

### 15.1 Container Internals and Image Engineering
- Container runtime stack (OCI spec, `containerd`, `runc`, Linux namespaces + cgroups + seccomp/AppArmor)
- Copy-on-write image layers, layer caching, multi-stage builds, and distroless/minimal base images
- Container networking (veth pairs, Linux bridges, CNI plugins, DNS resolution)
- Ephemeral vs persistent container storage, volumes, and bind mounts
- Non-root execution, read-only root filesystems, and resource request/limit tuning

### 15.2 Kubernetes Architecture and Internals
- Control plane internals (`kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`)
- Node components (`kubelet`, `kube-proxy`, CRI, CNI, CSI)
- Pod lifecycle, init containers, sidecars, and startup/readiness/liveness probes
- Workloads (`Deployment`, `StatefulSet`, `DaemonSet`, `Job`, `CronJob`) and Pod Disruption Budgets (`PDB`)
- Kubernetes networking (`ClusterIP`, `NodePort`, `LoadBalancer`, `Ingress`, Gateway API, `iptables` vs `IPVS` vs `eBPF`/Cilium)
- Scheduling mechanics (taints, tolerations, node/pod affinity, topology spread constraints, priority/preemption)
- Autoscaling (`HPA`, `VPA`, `KEDA` event-driven scaling, Cluster Autoscaler / Karpenter)
- Custom Resource Definitions (CRDs), reconciliation loops, and Kubernetes Operators (Spark Operator, Flink Operator, Strimzi)

### 15.3 Cloud Architecture, IaC, and CI/CD
- Cloud networking primitives (VPC, CIDR planning, transit gateways, NAT gateways, egress costs)
- IAM roles, instance profiles, workload identity (IRSA/OIDC federation), and cross-account trust
- Infrastructure as Code (Terraform state management, locking, drift detection, modular composition)
- CI/CD pipelines (build caching, artifact promotion, immutable tags, GitOps with ArgoCD/Flux)
- Deployment strategies (rolling updates, blue-green, canary, feature-flag progressive delivery)

### 15.4 Workflow and Platform Orchestration
- DAG-based data orchestration (Airflow / Dagster: scheduler internals, executors, backfill, dynamic task mapping, sensor scalability)
- Durable execution engines (Temporal: event history replay, determinism constraints, activities, workflows, task queues, child workflows)
- Platform engineering abstractions (self-service golden paths, internal developer platforms, declarative pipeline specs)

---

## 16. Testing, Verification, and Chaos Engineering

### 16.1 Application and Data Testing Pyramid
- Test-Driven Development (TDD) and behavioral vs structural testing
- Unit testing, test doubles (fakes, stubs, mocks, spies), and dependency isolation
- Integration testing with ephemeral dependencies (`Testcontainers`, local Postgres/Kafka/MinIO)
- Contract testing (consumer-driven contracts, OpenAPI/Protobuf/Schema Registry compatibility checks)
- Property-based testing (`Hypothesis`) and fuzzing for parsers, serializers, and state machines
- Data pipeline testing (idempotency checks, deterministic transforms, schema regression, reconciliation assertions)

### 16.2 Load, Performance, and Fault-Injection Testing
- Load vs stress vs soak vs spike testing methodologies
- Open-loop vs closed-loop load generators and coordinated omission
- Traffic shadowing / dark launches and diff comparison
- Chaos engineering (fault injection, packet loss/latency simulation, broker/node kills, clock skew testing, GameDays)
- Coverage metrics (line vs branch vs mutation testing) and meaningful quality gates

---

## 17. Observability, Performance Engineering, SRE, and FinOps

### 17.1 Observability Pillars and Telemetry Pipelines
- Structured logging (JSON logs, correlation IDs, log levels, PII redaction, cardinality discipline)
- Metrics architectures (push vs pull, Prometheus scraping, counters/gauges/histograms/native histograms, RED/USE methods)
- High-cardinality metrics pitfalls and aggregation/recording rules
- Distributed tracing (W3C Trace Context, spans, baggage, OpenTelemetry SDK/Collector, head-based vs tail-based sampling)
- Continuous profiling (eBPF/async profilers, CPU/memory/mutex flamegraphs)

### 17.2 Production Resilience and Traffic Management
- Load shedding, admission control, and adaptive concurrency limits (Little's Law, Vegas/Gradient algorithms)
- Retry storms, metastable failures, exponential backoff, full/decorrelated jitter, and retry budgets
- Circuit breakers (closed, open, half-open state transitions, failure thresholds)
- Bulkheads (thread-pool, semaphore, and tenant isolation)
- Request hedging and timeout budgets across call chains

### 17.3 Performance Engineering and Tail-Latency Optimization
- Latency percentiles (`P50`, `P95`, `P99`, `P999`), tail-at-scale amplification, and Little's Law
- Bottleneck isolation (CPU, memory bandwidth, disk IOPS/queue depth, network RTT/bandwidth, lock contention)
- Connection pool sizing, keep-alive tuning, and thread-pool queue bounds
- Serialization/allocation hot-path elimination and batching trade-offs

### 17.4 SRE, Incident Management, and FinOps
- SLIs, SLOs, SLAs, error budgets, and multi-window burn-rate alerting
- Alert fatigue reduction (symptom-based vs cause-based alerts, runbooks)
- Incident response roles, mitigation-first triage, blameless postmortems, and action-item tracking
- Disaster recovery (RPO, RTO, multi-AZ vs multi-region failover, automated backup restoration drills)
- Capacity planning (headroom modeling, load forecasting, bottleneck scaling curves)
- Cloud FinOps (compute right-sizing, spot/preemptible workloads for Spark/batch, cross-AZ/egress cost optimization, storage tiering, unit economics per request/GB)
