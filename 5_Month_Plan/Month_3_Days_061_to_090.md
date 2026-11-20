# Month 3 (Days 61–90): Distributed Systems Theory, Distributed DBs, Object Storage & Apache Kafka Internals

> **Curriculum Coverage:** `8.1–8.5`, `9.1–9.3`, `10.1–10.3`, `13.1` (Consistent & Rendezvous Hashing)
> **Environment:** Main Laptop + Ubuntu 16 GB VM (3-node `etcd`, 3-broker KRaft Kafka, Schema Registry, MinIO, Postgres) + Azure Blob Storage Free Tier

---

### Day 61 — Network Models, FLP, CAP/PACELC & Physical Clock Limits (`8.1`)
1. **Pre-Requisite Learning Resource:**
   - *DDIA* Chapter 8 (*The Trouble with Distributed Systems*: Unreliable Networks, Unreliable Clocks, Process Pauses, Knowledge/Truth/Lies, pp. 273–304) + Daniel Abadi — *PACELC* (`cs-www.cs.yale.edu/homes/dna/papers/Abadi-pacelc.pdf`).
2. **Hands-On Activity:**
   - **What to build:** A fault-injecting distributed network simulator in Python (`src/dist_sim/network.py`) modeling asymmetric partitions, packet delays, process GC pauses, and skewed physical clocks.
   - **Functions & Components:**
     - `VirtualClock`: Each simulated node has independent clock drift (`skew_ms`, `rate_multiplier`) so Node A can be 120ms ahead of Node B or jump backward (NTP step!).
     - `SimulatedNetwork`: Supports `send(src, dst, msg)` with configurable `drop_prob`, `delay_range_ms`, `partition(group_a, group_b)` (full or asymmetric—A hears B, B drops A!), and `pause_node(node_id, duration_ms)` (simulating JVM GC pause / VM freeze).
     - `demonstrate_lww_data_loss()`: Two nodes concurrently write `key="x"` with `time.time()` timestamps; because Node B's clock is 200ms behind Node A, Node B's *newer* write gets silently dropped by Last-Write-Wins!
   - **Verify:** Run 100 iterations showing how physical timestamps silently lose causally newer updates under clock skew.
3. **Daily Split:**
   - **DSA (1 hr):** LC 200 *Number of Islands* & LC 695 *Max Area of Island* (DFS/BFS grid).
   - **HLD (30 min):** Classify Postgres (Single-leader), DynamoDB, Cassandra, and Spanner on the **CAP** and **PACELC** spectrum (Partition -> A vs C; Else -> Latency vs Consistency).
   - **LLD (30 min):** Design a `MonotonicClockGuard` detecting backward wall-clock jumps and clamping/failing safely (critical for Snowflake IDs and leases).

---

### Day 62 — Logical Time: Lamport Clocks, Vector Clocks & HLC (`8.1`)
1. **Pre-Requisite Learning Resource:**
   - Leslie Lamport — *Time, Clocks, and the Ordering of Events in a Distributed System* (§1–3) + *DDIA* Chapter 5 (*Detecting Concurrent Writes / Version Vectors*, pp. 184–192) + CockroachDB Blog on Hybrid Logical Clocks (HLC).
2. **Hands-On Activity:**
   - **What to build:** Implement **Lamport Clocks**, **Vector Clocks**, and **Hybrid Logical Clocks (HLC)** inside your simulator (`src/dist_sim/clocks.py`).
   - **Functions & Components:**
     - `LamportClock`: `tick()`, `send() -> int`, `receive(remote_ts)` updating `max(local, remote) + 1`. Prove: $a \to b \implies L(a) < L(b)$, while $L(a) < L(b)$ does NOT prove causality!
     - `VectorClock`: Stores `dict[node_id, int]`; implements `increment(node_id)`, `merge(other_vc)`, and `compare(vc1, vc2) -> BEFORE | AFTER | CONCURRENT | EQUAL`.
     - `HybridLogicalClock`: Stores `(physical_pt, logical_c)` combining wall clock with Lamport monotonicity so timestamps stay close to real time while preserving causality. *(Study Google TrueTime `[earliest, latest]` bounded uncertainty at `[T3]`.)*
   - **Verify:** Simulate a shopping cart across 3 replicas where Node A and Node B accept offline/partitioned writes—use `VectorClock.compare()` to detect `CONCURRENT` sibling conflict instead of silently overwriting!
3. **Daily Split:**
   - **DSA (1 hr):** LC 210 *Course Schedule II* & LC 310 *Minimum Height Trees* (topological peeling).
   - **HLD (30 min):** Why do Dynamo/Riak use **Dotted Version Vectors** instead of raw node-count vector clocks when clients reconnect to different coordinator nodes?
   - **LLD (30 min):** Design a `VersionedValueResolver` storing conflicting sibling values (`list[Sibling(vc, payload)]`) and pruning dominated ancestors on read-repair/merge.

---

### Day 63 — Consistency Models & CRDTs (`8.2`)
1. **Pre-Requisite Learning Resource:**
   - *DDIA* Chapter 9 (*Consistency and Consensus*: Linearizability, Ordering & Causality, pp. 321–351) + Marc Shapiro et al. (*Conflict-Free Replicated Data Types* overview: State-based CvRDT vs Op-based CmRDT).
2. **Hands-On Activity:**
   - **What to build:** Build runnable counterexamples & CRDTs (`src/dist_sim/consistency.py`) clarifying Linearizability vs Sequential vs Causal vs Eventual consistency.
   - **Functions & Components:**
     - `LinearizabilityChecker`: Records `(invoke_ts, return_ts, op, arg, result)` history across concurrent client threads and checks if a valid sequential total order respecting real-time overlap exists.
     - `GCounterCvRDT` & `PNCounterCvRDT` (State-based CRDT): `increment(node_id)`, `decrement(node_id)`, `value() -> int`, `merge(other_state)` (element-wise `max()`—associative, commutative, idempotent!).
     - `ORSetCvRDT` (Observed-Remove Set): Handles add+remove concurrency via unique element tags without resurrecting deleted items.
   - **Verify:** Deliver CRDT state merges out-of-order, duplicated, and across simulated partitions -> prove all replicas converge to the exact same state (`PNCounter` and `ORSet`)!
3. **Daily Split:**
   - **DSA (1 hr):** LC 399 *Evaluate Division* (weighted graph DFS/BFS) & LC 130 *Surrounded Regions*.
   - **HLD (30 min):** Clearly distinguish **Linearizability** (single-object real-time recency) vs **Serializability** (multi-object transaction isolation) vs **Strict Serializability** (`Spanner`) + **Client-Centric guarantees** (Read-Your-Writes, Monotonic Reads).
   - **LLD (30 min):** Design a `SessionTokenReadRouter` routing a user's `GET` to a read-replica *only* if `replica_lsn >= user_last_write_lsn` (guaranteeing Read-Your-Writes!).

---

### Day 64 — Replication: Single-Leader, Multi-Leader & Failover Hazards (`8.3`)
1. **Pre-Requisite Learning Resource:**
   - *DDIA* Chapter 5 (*Replication*: Leaders & Followers, Replication Lag Problems, Multi-Leader Write Conflicts, pp. 151–177).
2. **Hands-On Activity:**
   - **What to build:** Simulate Single-Leader (sync vs async) and Multi-Leader replication (`src/dist_sim/replication.py`) demonstrating failover data loss and split-brain divergence.
   - **Functions & Components:**
     - `LeaderNode`: Maintains `wal: list[LogEntry(epoch, offset, key, val)]`, `replicas`, `sync_mode` (`async` vs `semi_sync`).
     - `simulate_async_failover_data_loss()`: Client writes `offset=105` to Leader -> Leader replies OK -> Leader crashes *before* Follower pulls `105` -> Follower promotes at `epoch=2` -> old Leader rejoins with un-replicated `105` (must truncate conflicting suffix!).
     - `simulate_split_brain()`: Network partitions Leader from LB; LB promotes Follower to Leader 2 while old clients still hit Leader 1 -> diverged histories!
   - **Verify:** Demonstrate how monotonic **Leader Epoch + Fencing** stops the stale leader from accepting writes.
3. **Daily Split:**
   - **DSA (1 hr):** LC 133 *Clone Graph* & LC 785 *Is Graph Bipartite?* (2-coloring BFS).
   - **HLD (30 min):** Design a multi-region active-active Notes/Docs backend using Multi-Leader replication: how do you handle concurrent offline edits (OT vs CRDT vs conflict UI)?
   - **LLD (30 min):** Design a `ReplicationLogTruncator` reconciling a rejoined former leader's log against the new epoch's leader log.

---

### Day 65 — Leaderless / Dynamo Quorums, Hinted Handoff & Merkle Trees (`8.3`)
1. **Pre-Requisite Learning Resource:**
   - DeCandia et al. — *Dynamo: Amazon's Highly Available Key-Value Store* (§4.1–4.7: Partitioning, Versioning, `N/R/W`, Hinted Handoff, Merkle Trees) + *DDIA* Chapter 5 (*Leaderless Replication*, pp. 177–199).
2. **Hands-On Activity:**
   - **What to build:** A **Dynamo-Style Leaderless KV Cluster** simulator (`src/dist_sim/dynamo.py`) with $(N, W, R)$ quorums, Read Repair, Hinted Handoff, and Merkle-Tree Anti-Entropy.
   - **Functions & Components:**
     - `DynamoCoordinator`: Routes `put(key, val)` to top $N$ healthy preference-list nodes; succeeds when $\ge W$ ack. Routes `get(key)` to $N$ nodes, waits for $R$ replies, returns latest version(s), triggers **async Read Repair** to stale responders.
     - `SloppyQuorum & HintedHandoff`: When a primary node is unreachable, writes to a fallback node with `hint_target=dead_node`, draining hints back when `dead_node` recovers.
     - `MerkleTree`: Binary hash tree over key-range buckets (`build_tree(kv_items)`, `diff_trees(tree_a, tree_b) -> list[stale_bucket_ids]`) syncing only differing buckets instead of transferring the whole DB!
   - **Break Intentionally:** Set $N=3, W=1, R=1$ -> observe stale reads; set $W=2, R=2$ -> prove quorum intersection returns latest write (unless concurrent or sloppy quorum overlapped!).
3. **Daily Split:**
   - **DSA (1 hr):** LC 1631 *Path With Minimum Effort* & LC 778 *Swim in Rising Water* (Dijkstra / binary search + BFS).
   - **HLD (30 min):** Why does $W + R > N$ **not** guarantee linearizability by itself under concurrent writes or sloppy quorums?
   - **LLD (30 min):** Design a `MerkleRangeSyncer` dividing `[0, 2^32)` into $1024$ buckets and exchanging only mismatched leaf ranges between two replicas.

---

### Day 66 — Consistent Hashing (VNodes, Bounded Load) & Rendezvous Hashing (`9.1`, `13.1`)
1. **Pre-Requisite Learning Resource:**
   - Karger et al. — *Consistent Hashing and Random Trees* (core ring concept) + Google Research Blog (*Consistent Hashing with Bounded Loads* — Mirrokni et al.) + Wikipedia/paper summary on **Rendezvous (HRW) Hashing**.
2. **Hands-On Activity:**
   - **What to build:** Implement and benchmark **Modulo Hashing vs Consistent Hashing (with Virtual Nodes) vs Bounded-Load Consistent Hashing vs Rendezvous Hashing** (`src/dist_sim/hashing.py`).
   - **Functions & Components:**
     - `ConsistentHashRing(vnodes=150)`: Uses `hashlib.md5`/`sha256` -> `int` ring + sorted list + `bisect.bisect_right` for $O(\log(N \cdot V))$ node lookup; `add_node`, `remove_node`, `get_node(key)`, `get_preference_list(key, n)` (skipping duplicate physical nodes!).
     - `BoundedLoadConsistentHashRing(epsilon=1.25)`: Tracks current load per node; if target node exceeds `ceil((1 + epsilon) * avg_load)`, probes clockwise to next available node.
     - `RendezvousHasher`: Computes `score(node, key)` and picks `argmax` (zero ring state needed!).
   - **Verify:** Place 100,000 keys across 10 nodes; remove 1 node and measure: (a) % keys moved (`~10%` vs `~90%` for `hash % N`!), and (b) standard deviation of keys per node with `vnodes=1` vs `vnodes=200`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 528 *Random Pick with Weight* (`bisect` prefix sums) & LC 380 *Insert Delete GetRandom O(1)*.
   - **HLD (30 min):** How does a Discord/Uber stateful routing layer route 1M WebSocket/cache shards using Consistent Hashing with Bounded Loads to survive a viral hot-shard?
   - **LLD (30 min):** Design `ShardRouter` wrapping `ConsistentHashRing` with virtual-node weight heterogeneity (a 64 GB node gets 2x vnodes of a 32 GB node).

---

### Day 67 — Consensus Part 1: Raft Leader Election & Safety (`8.4`)
1. **Pre-Requisite Learning Resource:**
   - Diego Ongaro & John Ousterhout — *In Search of an Understandable Consensus Algorithm (Raft)* (§1–§5.4: Basics, Leader Election, Log Replication, Safety) + *The Secret Lives of Data* interactive Raft walkthrough (`thesecretlivesofdata.com/raft/`).
2. **Hands-On Activity:**
   - **What to build:** Part 1 of a **Pure-Python Raft Consensus Core** (`src/dist_sim/raft.py`) implementing terms, randomized election timeouts, `RequestVote` RPC, and split-vote recovery.
   - **Functions & Components:**
     - `RaftState` (`FOLLOWER`, `CANDIDATE`, `LEADER`), `current_term`, `voted_for`, `log: list[LogEntry(term, index, cmd)]`.
     - `on_election_timeout()`: Increments `current_term`, transitions to `CANDIDATE`, votes for self, sends `RequestVote(term, candidate_id, last_log_index, last_log_term)` to peers with randomized timeout `[150ms, 300ms]`.
     - `handle_request_vote(req)`: Rejects if `req.term < current_term`, checks `voted_for is None or == candidate_id`, AND enforces the **Raft up-to-date log restriction** (`last_log_term` higher, or equal term and `last_log_index >= mine`!).
   - **Break Intentionally:** Wire 5 `RaftNode` instances on your simulated network -> kill the Leader -> verify remaining 4 elect a new Leader in 1–2 randomized terms; partition into `[2 nodes] | [3 nodes]` -> prove minority `[2]` **never** elects a leader!
3. **Daily Split:**
   - **DSA (1 hr):** LC 787 (re-implement cleanly) & LC 1192 *Critical Connections in a Network* (Tarjan's bridges—finding single-point-of-failure edges in network topology!).
   - **HLD (30 min):** Compare Raft vs Multi-Paxos vs ZooKeeper ZAB (`[T3]`): why does Raft forbid committing entries from *previous* terms merely by counting replicas (Figure 8 safety bug)?
   - **LLD (30 min):** Design `RaftPersistentStorage` ensuring `current_term`, `voted_for`, and `log` are `fsync`-persisted *before* replying to any RPC.

---

### Day 68 — Consensus Part 2: Raft Log Replication, Snapshotting & Membership (`8.4`)
1. **Pre-Requisite Learning Resource:**
   - Ongaro & Ousterhout — *Raft Paper* (§5.3–§5.5, §6 Cluster Membership Changes, §7 Log Compaction) + *mit-pdos/6.5840* Raft Lab structure notes (`pdos.csail.mit.edu/6.824/labs/lab-raft.html`).
2. **Hands-On Activity:**
   - **What to build:** Complete your Python Raft node (`src/dist_sim/raft.py`) with `AppendEntries` heartbeats, log conflict backtracking, `commit_index` advancement, and snapshot compaction.
   - **Functions & Components:**
     - `handle_append_entries(req)`: Resets election timer if `req.term >= current_term`; enforces `prev_log_index`/`prev_log_term` match; truncates conflicting follower suffix and appends new entries; advances `commit_index = min(req.leader_commit, last_new_index)`.
     - `leader_replicate_tick()`: Sends `AppendEntries` per peer using `next_index[peer]`, decrements `next_index` on rejection, advances `match_index` on success, commits entry $N$ once majority `match_index >= N` **and** `log[N].term == current_term`.
     - `take_snapshot(last_included_index)`: Discards log prefix up to `last_included_index`, stores `(last_included_index, last_included_term, state_bytes)`.
   - **Verify:** Partition Leader with 1 uncommitted entry, commit 3 entries on the majority side, heal partition -> verify stale Leader steps down on higher term and overwrites its uncommitted entry to match majority!
3. **Daily Split:**
   - **DSA (1 hr):** LC 207 / LC 210 + LC 685 *Redundant Connection II* (directed graph tree restoration).
   - **HLD (30 min):** Why must Raft membership changes add/remove 1 server at a time (or use Joint Consensus $C_{\text{old,new}}$) instead of jumping straight from 3 nodes to 5 nodes simultaneously?
   - **LLD (30 min):** Design `InstallSnapshotRPC` chunked transfer for a lagging follower whose `next_index` has already been compacted by the leader.

---

### Day 69 — Distributed Locks, Leases, Fencing Tokens & `etcd` (`8.4`)
1. **Pre-Requisite Learning Resource:**
   - Martin Kleppmann — *How to do distributed locking* (`martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html` — GC pause & Fencing Tokens!) + `etcd` Official Docs (*Learning: Concurrency & Leases*) + SWIM / Phi Accrual Failure Detector summaries (`[T3]`).
2. **Hands-On Activity:**
   - **What to build:** Stand up a **3-node `etcd` cluster** in Docker on Ubuntu VM (`mem_limit: 256m` per node) + a Python worker/storage test proving why leases alone are unsafe without fencing tokens (`src/dist_sim/etcd_fencing.py`).
   - **Functions & Components:**
     - `acquire_lease_lock(etcd_client, lock_name, ttl_sec=5) -> int`: Creates `etcd` lease + transaction `If(create_revision == 0).Then(Put(lock, owner, lease))`, returning `mod_revision` as the **monotonic Fencing Token**!
     - `FencedStorageService`: Rejects any `write(key, val, fencing_token)` if `fencing_token < last_seen_token[key]`.
     - `demonstrate_gc_pause_safety()`: Worker 1 acquires lock (`token=10`), sleeps 8s (lease expires!), Worker 2 acquires lock (`token=11`) and writes `token=11` to storage -> Worker 1 wakes up and tries to write with `token=10` -> `FencedStorageService` rejects Worker 1!
   - **Verify:** Kill 1 of 3 `etcd` containers (`docker stop etcd2`) -> locks still work (2/3 quorum); kill 2 of 3 -> locks block cleanly preserving safety!
3. **Daily Split:**
   - **DSA (1 hr):** LC 1584 *Min Cost to Connect All Points* & LC 1631 (re-solve cleanly).
   - **HLD (30 min):** Explain Kleppmann's critique of Redis `Redlock` and why a **ZooKeeper/`etcd` leader election + Fencing Token** is required whenever lock correctness protects data invariants. Also contrast SWIM gossip failure detection vs Phi Accrual Failure Detector (`[T3]`).
   - **LLD (30 min):** Design an `etcd`-style `LeaseManager` (`grant(ttl)`, `keepalive(lease_id)`, `revoke(lease_id)`, `attach_key(key, lease_id)`) expiring attached keys in $O(\log N)$ via min-heap.

---

### Day 70 — Distributed Transactions: 2PC, Saga, TCC & Transactional Outbox (`8.5`)
1. **Pre-Requisite Learning Resource:**
   - *DDIA* Chapter 9 (*Atomic Commit and Two-Phase Commit (2PC)*, pp. 354–360) + Chris Richardson (`microservices.io/patterns/data/transactional-outbox.html` & `saga.html`) + Calvin/Percolator high-level summaries (`[T3]`).
2. **Hands-On Activity:**
   - **What to build:** Build an **Order -> Payment -> Inventory Saga Orchestrator** AND a **Transactional Outbox + Idempotent Inbox** (`src/dist_sim/saga_outbox.py`) backed by Postgres on Ubuntu VM.
   - **Functions & Components:**
     - `OutboxRelay`: Polls `outbox(id, aggregate_id, event_type, payload, published_at)` using `FOR UPDATE SKIP LOCKED`, publishes event, marks `published_at`.
     - `InboxDedupTable`: `processed_events(event_id PK, processed_at)` inside consumer's local transaction (`INSERT ... ON CONFLICT DO NOTHING` before mutating state).
     - `OrderSagaOrchestrator`: Runs forward steps `ReserveInventory` -> `ChargePayment` -> `ConfirmOrder`; if `ChargePayment` fails, runs compensating action `ReleaseInventory` (must be **idempotent & retriable**, using semantic lock status `PENDING_CANCEL`!).
   - **Break Intentionally:** Crash the `OutboxRelay` after publishing to downstream but before updating `published_at` -> verify relay republishes AND `InboxDedupTable` prevents double-execution!
3. **Daily Split:**
   - **DSA (1 hr):** LC 1462 *Course Schedule IV* & LC 851 *Loud and Rich* (graph reachability / topo DP).
   - **HLD (30 min):** Why does Two-Phase Commit (2PC) block forever if the Coordinator crashes after `PREPARE = YES`? Compare Saga **Choreography** (event-driven, cyclic risk) vs **Orchestration** (explicit state machine) vs **TCC (Try-Confirm-Cancel)**.
   - **LLD (30 min):** Design a persistent `SagaStateMachine` (`step_index`, `status`, `compensations_stack`, `resume_from_crash()`).

---

### Day 71 — Distributed NoSQL Internals: Cassandra & DynamoDB (`9.1`)
1. **Pre-Requisite Learning Resource:**
   - Discord Engineering Blog — *How Discord Stores Trillions of Messages* (`discord.com/blog/how-discord-stores-trillions-of-messages` — Cassandra -> ScyllaDB partitioning, tombstones, hot partitions!) + USENIX ATC '22 *Amazon DynamoDB: A Scalable, Predictably Performant, and Fully Managed NoSQL Database Service* (§2–4).
2. **Hands-On Activity:**
   - **What to build:** Run single-node **ScyllaDB** or **Cassandra** in Docker on Ubuntu VM (`--smp 1 --memory 1G`) OR model CQL tables and reproduce **wide-partition & tombstone death** (`src/nosql_lab/cassandra_lab.py`).
   - **Functions & Components:**
     - Create `messages_by_channel(channel_id TEXT, bucket INT, message_id TIMEUUID, body TEXT, PRIMARY KEY ((channel_id, bucket), message_id)) WITH CLUSTERING ORDER BY (message_id DESC)`.
     - Observe why partitioning by `channel_id` alone creates huge multi-GB hot partitions, and how adding a time-bucket `(channel_id, bucket)` bounds partition size!
     - **Break It:** Insert 50,000 rows in one partition and `DELETE` 49,999 of them -> run `SELECT *` with tracing (`TRACING ON`) and observe thousands of **tombstones** scanned!
   - **Verify:** Clean up container (`docker compose down`) after inspecting tracing and data modeling.
3. **Daily Split:**
   - **DSA (1 hr):** LC 215 & LC 373 *Find K Pairs with Smallest Sums* (Heap $K$-way merge).
   - **HLD (30 min):** Contrast **Local Secondary Index (document-partitioned)** (fast writes, scatter-gather reads) vs **Global Secondary Index (term-partitioned)** (fast reads, cross-partition async/distributed writes).
   - **LLD (30 min):** Design a `TimeBucketedPartitionKeyResolver` dynamically splitting high-volume channels across `N` sub-buckets based on volume tier.

---

### Day 72 — NewSQL & Distributed SQL: Spanner, CockroachDB & Distributed Joins (`9.2`)
1. **Pre-Requisite Learning Resource:**
   - Corbett et al. — *Spanner: Google's Globally-Distributed Database* (§1–4: Paxos groups, TrueTime, Lock-free read-only transactions) + CockroachDB Docs (*Architecture Overview: SQL -> KV Mapping, Raft Ranges, HLC Read Restarts*).
2. **Hands-On Activity:**
   - **What to build:** Run a local **3-node CockroachDB** cluster (insecure single-machine Docker Compose, `cache=.25GiB` per node) on Ubuntu VM (`src/nosql_lab/crdb_lab.py`).
   - **Functions & Components:**
     - Create a table, inspect its automatic **Raft range splits** (`SHOW RANGES FROM TABLE ...`), and inspect how SQL rows map to ordered KV keys (`SHOW EXPERIMENTAL_FINGERPRINTS` / `EXPLAIN (DISTSQL)`).
     - Compare **Broadcast Join** vs **Distributed Shuffle Join** in `EXPLAIN` across ranges.
     - Test multi-region survivor/follower-read concepts (`AS OF SYSTEM TIME follower_read_timestamp()`).
   - **Break Intentionally:** Stop 1 of the 3 CockroachDB nodes -> queries continue succeeding cleanly via Raft quorum (2/3); stop 2 of 3 -> writes stall preserving consistency; tear down afterward (`docker compose down`).
3. **Daily Split:**
   - **DSA (1 hr):** LC 407 *Trapping Rain Water II* (2D min-heap priority queue boundary BFS).
   - **HLD (30 min):** Explain how Spanner achieves **External Consistency (Strict Serializability)** across continents using TrueTime `[earliest, latest]` commit-wait vs CockroachDB's HLC read restarts without atomic clocks.
   - **LLD (30 min):** Design a `RangeSplitter` mapping primary-key ranges `[start_key, end_key)` to Raft group IDs and dynamically splitting any range exceeding 64 MB.

---

### Day 73 — Event-Driven Architecture, Event Sourcing & Schema Registry (`10.1`)
1. **Pre-Requisite Learning Resource:**
   - Martin Fowler — *What do you mean by "Event-Driven"?* (`martinfowler.com/articles/201701-event-driven.html`: Event Notification vs Event-Carried State Transfer vs Event Sourcing) + Confluent Docs (*Schema Evolution and Compatibility*).
2. **Hands-On Activity:**
   - **What to build:** Bring up **1-broker KRaft Kafka + Schema Registry** on Ubuntu VM (`mem_limit` capped) and build an Event Sourcing + Avro Schema Compatibility lab (`src/kafka_lab/schema_lab.py`).
   - **Functions & Components:**
     - `EventSourcedAccount`: Reconstructs account balance solely by replaying `AccountOpened`, `MoneyDeposited`, `MoneyWithdrawn` events from an append-only list + periodic **snapshot** at event version $V$.
     - `SchemaRegistryClient`: Registers Avro schema `v1` (`user_id`, `amount`), inspects the 5-byte Kafka wire header (`0x00` magic byte + 4-byte schema ID!), and tests registering `v2`:
       - Adding an optional field with default -> passes `BACKWARD`, `FORWARD`, `FULL`.
       - Adding a required field *without* default -> **rejected** (`409 Incompatible schema`)!
   - **Verify:** Inspect raw serialized bytes to confirm magic byte + 4-byte schema ID prefix before payload.
3. **Daily Split:**
   - **DSA (1 hr):** LC 252 *Meeting Rooms* & LC 1851 *Minimum Interval to Include Each Query* (Sweep line + Min-Heap).
   - **HLD (30 min):** Contrast **Event Notification** (tiny `{order_id}` -> forces callback read), **Event-Carried State Transfer** (full payload -> local cache autonomy), and **CQRS + Event Sourcing** (immutable event log + materialized read views).
   - **LLD (30 min):** Design an `EventSourcedAggregate` base class supporting `apply(event)`, `uncommitted_events`, optimistic concurrency `expected_version`, and snapshot rehydration.

---

### Day 74 — Kafka Broker Internals: Partitions, `.log`/`.index` Segments & KRaft (`10.2`)
1. **Pre-Requisite Learning Resource:**
   - Apache Kafka Official Docs — *Design: Implementation* (Filesystem/Pagecache/Sendfile) & *KRaft Overview* (KIP-500) + Jay Kreps — *The Log: What every software engineer should know about real-time data's unifying abstraction* (Parts 1–2).
2. **Hands-On Activity:**
   - **What to build:** Spin up a **3-broker KRaft Kafka cluster** on Ubuntu VM (`KAFKA_HEAP_OPTS="-Xms384m -Xmx512m"` per broker). Inspect the raw storage engine files inside a broker container (`src/kafka_lab/broker_inspect.sh`).
   - **Functions & Components:**
     - Create topic `orders` with `partitions=6`, `replication-factor=3`, `segment.bytes=1048576` (1 MB segments so segments roll visibly!).
     - Produce 50k messages and `docker exec` into `/var/lib/kafka/data/orders-0/`:
       - Inspect `00000000000000000000.log`, `.index` (sparse offset -> physical byte position), and `.timeindex` (timestamp -> offset).
       - Run `kafka-dump-log.sh --files ...log --print-data-log` to read raw record batches, base offsets, CRCs, and producer IDs!
     - Create a second topic with `cleanup.policy=compact`, produce updates + `None` tombstones for the same key, trigger compaction, and verify old values vanish while tombstones persist until `delete.retention.ms`.
   - **Verify:** Observe `kafka-metadata-quorum.sh describe --status` showing the KRaft Raft quorum leader & voters without ZooKeeper!
3. **Daily Split:**
   - **DSA (1 hr):** LC 729 *My Calendar I* & LC 731 *My Calendar II* (interval boundary counting / BST).
   - **HLD (30 min):** Explain how Kafka finds offset `4,123,456` in $O(\log S)$ without scanning gigabytes of `.log` files (binary search filename -> binary search sparse `.index` -> sequential scan inside 4 KB block).
   - **LLD (30 min):** Design a `LogSegmentManager` (`append(record)`, `roll_new_segment()`, `lookup_offset(target_offset)`) with sparse indexing every 4 KB written.

---

### Day 75 — Kafka Replication, ISR, High Watermark & Leader Epochs (`10.2`)
1. **Pre-Requisite Learning Resource:**
   - Apache Kafka Docs — *Design: Replication* (ISR, Unclean Leader Election, Committed Messages) + KIP-101 / Confluent Blog (*Kafka Replication & Leader Epochs* explaining data loss/divergence fixed by Leader Epoch).
2. **Hands-On Activity:**
   - **What to build:** A hands-on durability & replication failure lab on your 3-broker KRaft cluster (`src/kafka_lab/replication_lab.py`).
   - **Functions & Components:**
     - Topic `critical_payments`: `RF=3`, `min.insync.replicas=2`, `unclean.leader.election.enable=false`.
     - Run `kafka-topics.sh --describe` -> note `Leader`, `Replicas`, and `Isr` for each partition.
     - Producer script writing 1 msg/sec with `acks=all` logging `(offset, partition, success/error)`.
   - **Break Intentionally:**
     1. Stop 1 follower broker (`docker stop kafka-2`) -> watch `Isr` shrink from `[1,2,3]` to `[1,3]` while `acks=all` producer *keeps succeeding*.
     2. Stop a 2nd broker (`docker stop kafka-3`) -> `Isr` is `[1] < min.insync.replicas=2` -> producer immediately fails with `NOT_ENOUGH_REPLICAS` (trading availability for **zero data loss**!).
     3. Restart brokers and watch `Isr` catch up via High Watermark & Leader Epoch truncation.
3. **Daily Split:**
   - **DSA (1 hr):** LC 239 *Sliding Window Maximum* (monotonic decreasing deque in $O(n)$).
   - **HLD (30 min):** Walk through why **High Watermark** truncation caused silent data loss or log divergence before **Leader Epoch (KIP-101)**, and explain the exact trade-off of `unclean.leader.election.enable=true`.
   - **LLD (30 min):** Design an `ISRTracker` on a partition leader advancing `high_watermark = min(follower_leo for follower in isr)` and shrinking `isr` when `now - last_fetch_ts > replica_lag_time_max`.

---

### Day 76 — Kafka Producers: Batching, Compression, Partitioners & Idempotence (`10.3`)
1. **Pre-Requisite Learning Resource:**
   - Apache Kafka Docs — *Producer Configs* (`acks`, `linger.ms`, `batch.size`, `compression.type`, `enable.idempotence`, `max.in.flight.requests.per.connection`) + KIP-98 (*Idempotent & Transactional Producer*).
2. **Hands-On Activity:**
   - **What to build:** Benchmark & prove Kafka Producer batching, ordering, and idempotence (`src/kafka_lab/producer_lab.py` using `confluent-kafka`).
   - **Functions & Components:**
     - `benchmark_batching()`: Sends 100k events comparing `(linger.ms=0, batch.size=16384, compression='none')` vs `(linger.ms=20, batch.size=131072, compression='lz4'/'zstd')`, reporting throughput (msgs/sec) and broker disk bytes consumed.
     - `CustomTenantPartitioner`: Routes `tenant_id` keys via deterministic Murmur2/CRC32 hash, while supporting key-salting for a noisy mega-tenant!
     - `idempotence_config`: Configures `enable.idempotence=True`, `acks='all'`, `max.in.flight.requests.per.connection=5` (how Kafka preserves strict per-partition ordering even with 5 pipelined batches!).
   - **Verify:** Observe how `enable.idempotence=True` assigns `(PID, Epoch, SequenceNumber)` so broker-side deduplication discards retried batches without reordering.
3. **Daily Split:**
   - **DSA (1 hr):** LC 853 *Car Fleet* & LC 907 *Sum of Subarray Minimums* (monotonic stack).
   - **HLD (30 min):** How does a producer `RecordAccumulator` buffer batches per partition, apply `max.block.ms` backpressure when `buffer.memory` fills, and why does `zstd`/`lz4` batch compression dramatically boost broker & network throughput?
   - **LLD (30 min):** Design a `PartitionedBatchAccumulator` (`append(partition, record)`, `drain_ready_batches(linger_ms, batch_size)`, `enforce_memory_budget()`).

---

### Day 77 — Kafka Consumers, Rebalance Protocols & Rebalance Storms (`10.3`)
1. **Pre-Requisite Learning Resource:**
   - Confluent Blog / KIP-429 (*Incremental Cooperative Rebalancing*) & KIP-848 (*Next Generation Consumer Rebalance Protocol*) + Kafka Docs (*Consumer Configs*: `heartbeat.interval.ms`, `session.timeout.ms`, `max.poll.interval.ms`).
2. **Hands-On Activity:**
   - **What to build:** A multi-consumer group laboratory (`src/kafka_lab/consumer_lab.py`) observing partition assignments, offset commits (`__consumer_offsets`), and rebalance storms.
   - **Functions & Components:**
     - Run 3 consumer processes in group `orders-workers` consuming a 6-partition topic (`partition.assignment.strategy='cooperative-sticky'`), logging `on_assign` and `on_revoke` callbacks.
     - Disable `enable.auto.commit=False`; commit offsets explicitly *only after* processing completes (At-Least-Once!)—prove why auto-commit loses messages on crash!
   - **Break Intentionally:**
     1. Contrast `range` (eager stop-the-world!) vs `cooperative-sticky` when adding a 4th consumer (only 1–2 partitions move, others keep processing).
     2. Set `max.poll.interval.ms=5000` and inject a `7s` slow DB call -> watch the consumer get kicked out of the group, trigger a **Rebalance Storm**, and cause duplicate message processing on another consumer!
3. **Daily Split:**
   - **DSA (1 hr):** LC 496 *Next Greater Element I* & LC 503 *Next Greater Element II* (circular monotonic stack).
   - **HLD (30 min):** Explain the difference between Kafka's **heartbeat thread** (`session.timeout.ms`) and **processing loop** (`max.poll.interval.ms`), and how decoupling polling from slow worker threads (or pausing partitions via `pause()`/`resume()`) prevents rebalance storms.
   - **LLD (30 min):** Design a `CooperativeStickyAssignor` algorithm (`assign(members, topic_partitions)`) minimizing partition movement when 1 member joins or leaves.

---

### Day 78 — Kafka Transactions (EOS), Dead-Letter Topics & Poison Pills (`10.3`)
1. **Pre-Requisite Learning Resource:**
   - Confluent / Apache Kafka Docs (*Transactions in Apache Kafka* + *Read Committed Isolation*) + Uber Engineering Blog (*Building Reliable Reprocessing and Dead Letter Queues with Kafka*).
2. **Hands-On Activity:**
   - **What to build:** Build an **Exactly-Once Consume-Transform-Produce** pipeline (`init_transactions`, `send_offsets_to_transaction`) AND a **Non-Blocking Retry + Dead-Letter Topic (DLT)** handler (`src/kafka_lab/eos_dlt.py`).
   - **Functions & Components:**
     - `TransactionalProcessor`: Reads from `raw_orders`, transforms, produces to `enriched_orders` AND commits input offsets atomically inside `begin_transaction()` -> `commit_transaction()` (or `abort_transaction()` on error); verify downstream consumer with `isolation.level='read_committed'` skips aborted markers!
     - `RetryAndDLTProcessor`: On transient failure, routes to `orders.retry.5s` / `orders.retry.30s` with retry-count header instead of blocking the partition! On permanent poison-pill (`JSONDecodeError` or `retry > 3`), sends to `orders.dlt` with original headers, error trace, and offset metadata.
   - **Break Intentionally:** Send a malformed poison-pill payload -> verify healthy subsequent messages are NOT blocked and the poison pill lands cleanly in `orders.dlt`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 901 *Online Stock Span* & LC 402 *Remove K Digits*.
   - **HLD (30 min):** Why do non-blocking Retry Topics trade **strict per-key ordering** for **partition availability**, and when must a financial ledger *block/pause* instead of skipping ahead?
   - **LLD (30 min):** Design `PoisonPillRouter` tracking retry attempts, error classification (`TRANSIENT` vs `FATAL`), and exponential delay topic selection.

---

### Days 79–82 — Object Storage Internals (MinIO + Azure Blob Storage) (`9.3`)

#### Day 79 — Object Storage Architecture, Multipart Uploads & Strong Consistency (`9.3`)
1. **Pre-Requisite Learning Resource:**
   - AWS S3 Architecture & Consistency Docs (*Strong Read-After-Write Consistency*, *Multipart Upload Overview*, *Performance Guidelines: Prefix Parallelization*) + Azure Blob Storage Docs (*Block Blobs & SAS Tokens*).
2. **Hands-On Activity:**
   - **What to build:** Run **MinIO** on Ubuntu VM (`mem_limit: 512m`) AND connect to your **Azure Blob Storage Free Tier** (`src/objstore_lab/upload_lab.py`).
   - **Functions & Components:**
     - `multipart_upload_verified(bucket, key, filepath, part_size_mb=5)`: Splits a 30 MB file into parts, uploads parts concurrently via `ThreadPoolExecutor`, tracks per-part MD5/ETags, calls `CompleteMultipartUpload`, and handles aborting stale incomplete parts (`AbortMultipartUpload` lifecycle).
     - `test_byte_range_parallel_read(bucket, key)`: Fetches `[0..5MB]`, `[5MB..10MB]` concurrently using HTTP `Range: bytes=...` headers—showing how Parquet/DuckDB read only column footers from object storage!
   - **Verify:** Inspect actual uploaded object metadata & ETag format on multipart vs single-PUT uploads.
3. **Daily Split:**
   - **DSA (1 hr):** LC 148 *Sort List* (merge sort on linked list in $O(1)$ space) & LC 25 *Reverse Nodes in k-Group*.
   - **HLD (30 min):** Explain why Object Storage is *not* a POSIX filesystem (flat namespace, no atomic `append` or directory `rename`, metadata index vs blob store separation, prefix hashing for request-rate scaling).
   - **LLD (30 min):** Design a `ResumableChunkedUploader` persisting completed `(part_number, etag)` checkpoints locally so a 5 GB upload resumes after network drop without re-uploading completed parts.

#### Day 80 — Presigned URLs, Direct Client Offload & Lifecycle/WORM (`9.3`)
1. **Pre-Requisite Learning Resource:**
   - Azure Docs (*Grant limited access using Shared Access Signatures (SAS)*) + AWS S3 Docs (*Presigned URLs* & *Object Lock WORM*).
2. **Hands-On Activity:**
   - **What to build:** Add a **Direct-to-Object-Storage Offload** workflow to your FastAPI service (`src/objstore_lab/presign_api.py`) talking to MinIO and Azure Blob SAS.
   - **Functions & Components:**
     - `POST /v1/uploads/init`: Validates auth & file size/MIME type, creates `pending` metadata row in Postgres, generates a short-lived (60s) **Presigned PUT URL** (MinIO) / **SAS Token** (Azure Blob) with strict content-length/prefix constraints.
     - Client uploads binary directly to MinIO/Azure Blob **bypassing your FastAPI RAM/CPU completely**!
     - `POST /v1/uploads/{id}/complete`: Verifies object exists (`HEAD` request + ETag/size check) and transitions Postgres status to `ready`.
   - **Break Intentionally:** Try uploading a file larger than the signed constraint or after 60s expiry -> confirm object storage rejects it with `403 Forbidden`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 287 *Find the Duplicate Number* (Floyd's Tortoise & Hare cycle detection) & LC 138 *Copy List with Random Pointer*.
   - **HLD (30 min):** Design **Dropbox / Video Upload Pipeline**: why files NEVER stream through backend API servers, how chunk deduplication (content-addressed SHA-256 blocks) saves storage, and lifecycle tiering (Hot -> Cool -> Archive) + WORM compliance locks.
   - **LLD (30 min):** Design a `StorageLifecycleEvaluator` scanning object metadata `(age_days, access_tier, legal_hold, small_file_bytes)` and emitting transition/compaction/expiration actions.

#### Day 81 — Transactional Outbox + Kafka + Object Storage Integration (`8.5`, `9.3`, `10.3`)
1. **Pre-Requisite Learning Resource:**
   - Review your Day 70 Outbox + Day 76–78 Kafka notes.
2. **Hands-On Activity:**
   - **What to build:** Connect your Month 2 **Order & Wallet Service** -> **Postgres Transactional Outbox** -> **KRaft Kafka (`orders.events`)** -> **Object Storage Archiver Consumer** writing batched Parquet/JSONL files to MinIO/Azure Blob (`src/archiver/consumer.py`).
   - **Functions & Components:**
     - `OutboxKafkaRelay`: Polls Postgres `outbox` table, produces to Kafka keyed by `order_id` with `enable.idempotence=True` + `acks=all`, updates `published_at` after broker ACK.
     - `BatchArchiveConsumer`: Consumes `orders.events` in batches of 500 or every 5s, writes immutable batch file to MinIO/Azure Blob under `year=YYYY/month=MM/day=DD/part-<offset>.json`, commits Kafka offset **only after** `PutObject` returns 200 OK!
   - **Break Intentionally:** Kill the archiver consumer *after* `PutObject` succeeds and *before* committing Kafka offset -> verify replay produces deterministic/idempotent key overwrite rather than duplicate/missing data.
3. **Daily Split:**
   - **DSA (1 hr):** LC 105 (re-solve) & LC 114 *Flatten Binary Tree to Linked List*.
   - **HLD (30 min):** Walk through end-to-end delivery guarantees from API `POST /orders` -> Postgres -> Outbox -> Kafka -> S3/Azure Blob (where can duplicates happen, and why is zero-data-loss preserved?).
   - **LLD (30 min):** Design a `BatchFlushBuffer` flushing when `len >= max_records` OR `bytes >= max_bytes` OR `elapsed >= flush_interval_sec`.

#### Day 82 — Distributed Cache vs Distributed Lock vs Consensus Trade-Off Drill (`8.2`, `8.4`, `9.1`)
1. **Pre-Requisite Learning Resource:**
   - Jepsen Analyses summaries (`jepsen.io/analyses` — skim 2 short posts, e.g., etcd/Redis or MongoDB consistency bugs to understand how real DBs fail under partitions).
2. **Hands-On Activity:**
   - **What to build:** Write a **Jepsen-style Linearizability History Checker** (`src/dist_sim/jepsen_mini.py`) running concurrent client reads/writes against your Day 65 Dynamo simulator under $(N=3, W=1, R=1)$ vs $(N=3, W=2, R=2)$ vs `etcd` (`Raft`).
   - **Functions & Components:**
     - `ConcurrentClientHarness`: Spawns 5 threads issuing random `put(k, v)` and `get(k)` while logging exact `[start_ns, end_ns]` intervals.
     - `find_stale_read_violation(history)`: Detects when a read starting *after* a completed write returns an older value.
   - **Verify:** Observe frequent stale-read violations on $W=1, R=1$ and **0 violations** on `etcd`!
3. **Daily Split:**
   - **DSA (1 hr):** LC 236 *Lowest Common Ancestor of a Binary Tree* & LC 863 *All Nodes Distance K in Binary Tree*.
   - **HLD (30 min):** When should a system choose `etcd`/Spanner (CP/PC) vs Cassandra/DynamoDB (AP/PA+EL)? Give 2 concrete Uber examples for each.
   - **LLD (30 min):** Design a `FencingTokenValidatorMiddleware` for an object/file server rejecting stale-epoch writes and logging split-brain alerts.

---

### Days 83–90 — Kafka Failure Drills, Distributed Systems Synthesis & Month 3 Capstone

#### Day 83 — Kafka Partition Scaling, Skew & Consumer Lag Backpressure (`10.3`)
1. **Pre-Requisite Learning Resource:**
   - Confluent / LinkedIn Engineering Posts on Kafka Consumer Lag Monitoring & Partition Key Skew.
2. **Hands-On Activity:**
   - **What to build:** Simulate a **hot-partition key skew** and **consumer lag spike** on your 3-broker Kafka cluster (`src/kafka_lab/lag_lab.py`).
   - **Functions & Components:**
     - Produce 80% of events with `key="mega_tenant"` on a 6-partition topic -> verify `kafka-consumer-groups.sh --describe` shows Partition $P$ at 100k lag while 5 partitions sit idle at lag 0!
     - Implement **Key Salting (`f"{tenant_id}#{rand(0, 3)}"`)** or `RoundRobin` fallback for non-ordered tenants -> watch all 6 consumers parallelize cleanly.
     - Implement pause/resume backpressure: if consumer's local async DB batch queue is full, call `consumer.pause(partitions)` while still calling `consumer.poll()` so heartbeats continue without triggering a rebalance storm!
3. **Daily Split:**
   - **DSA (1 hr):** LC 329 *Longest Increasing Path in a Matrix* (DAG memoized DFS) & LC 778 (re-solve).
   - **HLD (30 min):** Why is increasing Kafka partition count *not* free (more open file handles, higher replication/leader election failover time, key-hash redistribution)?
   - **LLD (30 min):** Design a `BackpressureKafkaPump` decoupling `poll()` from bounded `asyncio.Queue` workers using `pause()` / `resume()` watermarks.

#### Day 84 — Raft & Consensus Failure Synthesis (`8.4`)
1. **Pre-Requisite Learning Resource:**
   - Review *Raft Paper* §5.4 (Safety) & §8 (Client Interaction — linearizable reads via `ReadIndex` / lease reads!).
2. **Hands-On Activity:**
   - **What to build:** Add **Linearizable Reads (`ReadIndex`)** to your Day 68 Python Raft core (`src/dist_sim/raft.py`) so a deposed partitioned Leader cannot serve stale reads!
   - **Functions & Components:**
     - `handle_client_read(key)`: Leader records `read_index = commit_index`, broadcasts heartbeat `AppendEntries` to confirm with a majority that it is *still* the leader in the current term, waits until `last_applied >= read_index`, and then reads local state machine!
   - **Verify:** Partition old Leader away from majority -> prove `handle_client_read()` on isolated old Leader fails/times out instead of serving stale data!
3. **Daily Split:**
   - **DSA (1 hr):** LC 417 & LC 269 (re-solve cleanly without notes).
   - **HLD (30 min):** Explain why serving `GET` directly from a Raft leader without a `ReadIndex` heartbeat round-trip (or bounded clock lease) violates linearizability during a network partition!
   - **LLD (30 min):** Design an `IdempotentClientSessionTable` inside Raft's state machine `(client_id, seq_num) -> cached_reply` preventing duplicate execution when a client retries a committed Raft command.

#### Day 85 — Distributed Unique ID Generators (`13.1`)
1. **Pre-Requisite Learning Resource:**
   - Twitter Snowflake original announcement / layout + RFC 9562 (*UUIDs*: UUIDv4 vs UUIDv7 time-ordered!) + ULID Spec (`github.com/ulid/spec`).
2. **Hands-On Activity:**
   - **What to build:** Implement a **Snowflake 64-bit ID Generator** AND a **UUIDv7** generator in Python (`src/primitives/id_gen.py`), then benchmark B-Tree Postgres insert speed of random UUIDv4 vs monotonic UUIDv7/Snowflake on 500k rows!
   - **Functions & Components:**
     - `SnowflakeGenerator(datacenter_id, worker_id)`: Layout `[1 sign bit | 41 timestamp ms | 5 dc | 5 worker | 12 sequence]`; handles `seq` overflow within same millisecond (busy-wait next ms) AND **clock regression** (refuses or waits if `now < last_ts`).
     - `benchmark_btree_insert_locality()`: Inserts 500k rows with `UUIDv4` PK vs `UUIDv7`/`Snowflake` PK into Postgres, measuring execution time, WAL bytes generated, and B-Tree index size (`pg_relation_size`)!
   - **Verify:** Observe massive B-Tree page-split/random-I/O slowdown with `UUIDv4` vs sequential right-edge appends with `UUIDv7`!
3. **Daily Split:**
   - **DSA (1 hr):** LC 535 *Encode and Decode TinyURL* & LC 271 *Encode and Decode Strings*.
   - **HLD (30 min):** Design a **Distributed Unique ID Service** generating 100k time-sortable 64-bit IDs/sec globally with zero coordination on the hot path.
   - **LLD (30 min):** Design `WorkerIdAllocator` using `etcd` ephemeral leases (`0..1023`) so restarting pods automatically claim unique worker IDs without collisions.

#### Day 86 — Multi-Region Architectures & Geo-Partitioning (`9.2`, `13.3`)
1. **Pre-Requisite Learning Resource:**
   - CockroachDB Docs (*Multi-Region Capabilities: Survival Goals & Table Localities*) + AWS Architecture Blog (*Cell-Based Architecture* `docs.aws.amazon.com/wellarchitected/latest/reducing-scope-of-impact-with-cell-based-architecture/`).
2. **Hands-On Activity:**
   - **What to build:** Model a **Cell-Based Multi-Region Routing Simulator** (`src/dist_sim/cells.py`) mapping users by home region (`US-EAST`, `EU-WEST`, `AP-SOUTH`) to isolated blast-radius cells.
   - **Functions & Components:**
     - `CellRouter`: Maps `tenant_id` -> `(region, cell_id)` via replicated global control-plane directory; enforces EU user data locality (`GDPR` pinning) and caps blast radius to 1 cell (<5% users) during outage.
     - `FailoverController`: Simulates regional outage (`RPO` data loss calculation for async replication vs `RTO` DNS/Anycast failover time).
3. **Daily Split:**
   - **DSA (1 hr):** LC 721 *Accounts Merge* (Union-Find + HashMap) & LC 947 *Most Stones Removed*.
   - **HLD (30 min):** Design a **Global Multi-Region Active-Active Ride/User Service**: compare Active-Passive (`RPO/RTO`) vs Regional-Home-Cell Active-Active (data locality, cross-region failover).
   - **LLD (30 min):** Design a `ControlPlaneDirectoryCache` (`cell_map`) designed to stay read-only and survive control-plane outages via local signed snapshots!

#### Day 87 — Designing Uber's Docstore / Schemaless & KV Storage (`9.1`, `9.2`)
1. **Pre-Requisite Learning Resource:**
   - Uber Engineering Blog — *Designing Schemaless, Uber's Scalable Datastore* (Parts 1 & 2) + *How Uber Built Docstore* (`uber.com/blog/schemaless-part-one/`).
2. **Hands-On Activity:**
   - **What to build:** Build an append-only **Schemaless Cell Store** prototype over Postgres (`src/nosql_lab/schemaless_mini.py`) modeling Uber's `(row_key, column_name, ref_key) -> immutable JSON body` shard model.
   - **Functions & Components:**
     - `ShardRouter`: Hashes `row_key` across 16 logical shards (mapped onto Postgres schemas/tables).
     - `put_cell(row_key, col_name, ref_key, body)`: Appends immutable version `(row_key, col_name, ref_key)` (`ON CONFLICT DO NOTHING`!) without overwriting old versions, and triggers async secondary index update.
     - `get_latest_cell(row_key, col_name)`: Reads highest `ref_key`.
   - **Verify:** Demonstrate idempotent retries and audit history without read-before-write locks.
3. **Daily Split:**
   - **DSA (1 hr):** LC 1244 *Design A Leaderboard* (or LC 355 re-solve) & LC 642 *Design Search Autocomplete System* (Trie + Heap).
   - **HLD (30 min):** Design a **Distributed Key-Value Store** (like Dynamo/Schemaless): cover consistent hashing, replication factor 3, tunable consistency, gossip membership, hinted handoff, and compaction.
   - **LLD (30 min):** Design an `AsyncSecondaryIndexUpdater` maintaining eventual consistency between shard primary tables and index shards.

#### Day 88 — Month 3 Chaos Drill: Breaking Kafka + Outbox + `etcd` (`8.4`, `8.5`, `10.2–10.3`)
1. **Pre-Requisite Learning Resource:**
   - Review your Month 3 `notes.md` + Kafka/Outbox logs.
2. **Hands-On Activity:**
   - **What to build:** Run a full-stack chaos sequence on your Ubuntu VM (Postgres + Outbox + 3-broker KRaft Kafka + Archiver -> MinIO):
     1. Kill Kafka Broker 2 while producing 100 events/sec -> verify zero dropped events (`acks=all, min.insync.replicas=2`).
     2. Pause Archiver consumer > `max.poll.interval.ms` -> observe rebalance -> recover cleanly via MinIO idempotent writes.
     3. Kill Outbox Relay mid-batch -> restart -> verify Inbox/Archiver deduplicates cleanly.
     4. Tear down all Month 3 Docker stacks (`docker compose down`) afterward to free Ubuntu VM RAM for Month 4 Flink & Spark!
3. **Daily Split:**
   - **DSA (1 hr):** LC 981 *Time Based Key-Value Store* + LC 1146 *Snapshot Array* (binary search on version history).
   - **HLD (30 min):** Design **Discord / Slack Real-Time Chat Backend** (WebSocket gateway servers, Consistent Hashing by `channel_id`, Kafka/PubSub fan-out, ScyllaDB time-bucketed storage).
   - **LLD (30 min):** Design a `GossipMembershipTable` (`node_id`, `heartbeat_counter`, `last_updated`, `state: ALIVE|SUSPECT|DEAD`) implementing SWIM/gossip merge rules.

#### Day 89 — Month 3 Whiteboarding Marathon: 3 Distributed Designs (`8.1–8.5`, `9.1–9.3`, `10.1–10.3`)
1. **Pre-Requisite Learning Resource:**
   - Review *DDIA* Ch. 5 & Ch. 9 key trade-off summaries.
2. **Hands-On Activity:**
   - **What to build:** No coding infrastructure today—do **two deep written Architectural Decision Records (ADRs)** from scratch:
     - **ADR 1:** Choosing PostgreSQL + Outbox + Kafka vs DynamoDB + Streams for an Order Ledger (justify consistency, operational overhead, failure modes, and cost).
     - **ADR 2:** Choosing `etcd` + Fencing Tokens vs Redis `SET NX PX` for preventing duplicate batch-billing runs.
3. **Daily Split:**
   - **DSA (1 hr):** Timed Mock: LC 23 *Merge k Sorted Lists* + LC 743 *Network Delay Time*.
   - **HLD (30 min):** Timed 30-min Blank-Page Design: **Distributed Lock & Coordination Service** (Chubby/`etcd` style: Raft quorum, leases, watches, fencing tokens, read linearizability).
   - **LLD (30 min):** Design a `WatchManager` notifying subscribed clients on prefix key changes (`/services/orders/*`) with monotonic `revision` resumption (no missed events on reconnect!).

#### Day 90 — Month 3 Comprehensive Assessment
1. **Pre-Requisite Learning Resource:**
   - Audit [Curriculum.md](./Curriculum.md) Sections `8.1–8.5`, `9.1–9.3`, `10.1–10.3`.
2. **Hands-On Activity:**
   - **What to build:** Clean-room 90-min practical on Ubuntu VM:
     1. Stand up 1-broker Kafka + Postgres Outbox from memory and demonstrate `acks=all` + `enable.idempotence` + consumer manual offset commit.
     2. Explain your Day 68 Raft `AppendEntries` + Figure-8 commit safety rule out loud without notes.
     3. Prune unused Docker volumes/images (`docker system df`) so 512 GB SSD stays clean for Month 4.
3. **Daily Split:**
   - **DSA (1 hr):** Timed Mock (unseen): LC 127 *Word Ladder* + LC 721 *Accounts Merge*.
   - **HLD (30 min):** Timed 30-min Blank-Page Design: **Uber Notification & Real-Time Event Ingestion Bus** (Kafka partitioning, Schema Registry, priority lanes, retry topics, DLT, object storage archive).
   - **LLD (30 min):** Timed 30-min Blank-Page Design: **Consistent Hashing Ring with Virtual Nodes & Bounded Load**.
