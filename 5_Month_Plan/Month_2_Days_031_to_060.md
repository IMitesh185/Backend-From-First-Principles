# Month 2 (Days 31–60): OS Memory/`epoll`, CPython Runtime/`asyncio`, Storage Engines, PostgreSQL Mastery, Caching & Queues

> **Curriculum Coverage:** `1.2`, `1.3`, `1.4`, `3.1–3.3`, `3.5` (profiling), `4.3` (idempotency), `4.4`, `5.1–5.4`, `6.1–6.3`, `7.1–7.4`, `16.1` (Testcontainers)
> **Environment:** Main Laptop + Ubuntu 16 GB VM (Docker Postgres 16, PgBouncer, Redis 7, Elasticsearch)

---

### Day 31 — Virtual Memory, Page Cache, `mmap` & Zero-Copy I/O (`1.2`)
1. **Pre-Requisite Learning Resource:**
   - *Operating Systems: Three Easy Pieces (OSTEP)* — Chapter 18 (*Paging: Introduction*) & Chapter 19 (*TLBs*) + Linux `man 2 mmap` & `man 2 sendfile`.
2. **Hands-On Activity:**
   - **What to build:** On Ubuntu 16 GB VM, run an OS page-cache & zero-copy benchmark suite (`src/os_lab/io_bench.py`) comparing buffered reads, `mmap`, `sendfile`, and `fsync` durability cost.
   - **Functions & Components:**
     - `generate_test_file(path, size_mb=512)`: Writes 512 MB binary payload.
     - `benchmark_buffered_read(path)` vs `benchmark_mmap_read(path)`: Reads sequentially and randomly, recording latency before and after `echo 3 | sudo tee /proc/sys/vm/drop_caches`.
     - `benchmark_fsync_cost(path, sync_every_write: bool)`: Writes 5,000 4 KB records with `os.fsync()` on every write vs batched every 1,000 writes.
     - `zero_copy_file_server(port, filepath)`: Uses `socket.sendfile()` to stream a 200 MB file without copying bytes through Python user space.
   - **Verify:** Watch `vmstat 1` (`bi`/`bo`, `cache`), `free -h`, and `strace -c` comparing read/write syscall counts vs `sendfile`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 199 *Binary Tree Right Side View* & LC 105 *Construct Binary Tree from Preorder and Inorder Traversal*.
   - **HLD (30 min):** Why can Apache Kafka serve 10 GB/s reads over the network using `sendfile` + OS Page Cache without caching messages inside the JVM heap?
   - **LLD (30 min):** Design a `PagedFileManager` class (`read_page(page_id) -> bytes`, `write_page(page_id, data_4kb)`, `flush()`) managing fixed 4 KB/8 KB pages and dirty flags.

---

### Day 32 — I/O Multiplexing: `select`, `poll`, `epoll` & `io_uring` (`1.3`)
1. **Pre-Requisite Learning Resource:**
   - Linux `man 7 epoll` (edge-triggered vs level-triggered, `EPOLLET`, `EAGAIN`) + Cloudflare Blog / LWN (*Efficient I/O with `io_uring`* overview) + Python Docs (`selectors` module).
2. **Hands-On Activity:**
   - **What to build:** A single-threaded concurrent TCP echo server using `selectors.EpollSelector` (`src/os_lab/epoll_server.py`) capable of holding 1,000+ concurrent idle/active connections.
   - **Functions & Components:**
     - `configure_nonblocking(sock)`: Calls `sock.setblocking(False)`.
     - `EpollEchoServer`: Registers listening socket (`EVENT_READ`), accepts new clients in loop until `BlockingIOError` (`EAGAIN`), maintains per-FD read/write byte buffers in a `dict[int, ClientBuffer]`, dynamically switches `EVENT_READ` / `EVENT_WRITE` registrations.
     - `stress_open_connections(host, port, n=1000)`: Opens 1,000 simultaneous client sockets and sends pings across them.
   - **Break Intentionally:** Raise `ulimit -n` too low (`ulimit -n 256`) and hit `EMFILE: Too many open files`; block inside the event loop with `time.sleep(2)` to see every socket stall.
3. **Daily Split:**
   - **DSA (1 hr):** LC 230 *Kth Smallest Element in a BST* & LC 124 *Binary Tree Maximum Path Sum*.
   - **HLD (30 min):** Explain why `select` ($O(N)$ FD copy + 1024 FD cap) and `poll` ($O(N)$ scan) collapse at 50k concurrent WebSocket connections while `epoll` ($O(\text{ready})$ interest/ready list) and `io_uring` (shared submission/completion rings, zero-syscall batching) scale.
   - **LLD (30 min):** Design a `ReactorEventLoop` (`register_fd`, `register_timer`, `run_once(timeout)`) bridging `epoll` readiness events and scheduled timer callbacks.

---

### Day 33 — Concurrency Primitives, CAS, Deadlocks & Work Stealing (`1.4`)
1. **Pre-Requisite Learning Resource:**
   - *OSTEP* — Chapter 28 (*Locks*: spinlocks, mutexes, Test-And-Set, Compare-And-Swap), Chapter 30 (*Condition Variables*), Chapter 31 (*Semaphores*), Chapter 32 (*Common Concurrency Problems*: deadlocks, priority inversion).
2. **Hands-On Activity:**
   - **What to build:** A multi-threaded concurrency laboratory (`src/conc_lab/primitives.py`) implementing a **Thread-Safe Bounded Blocking Queue** using `threading.Lock` + two `Condition` variables (`not_empty`, `not_full`).
   - **Functions & Components:**
     - `BoundedBlockingQueue[T]`:
       - `put(item: T, timeout: float | None = None) -> None`: Acquires lock, `while len == capacity` waits on `not_full` (MUST use `while`, never `if`, to guard against spurious wakeups!), appends item, calls `not_empty.notify()`.
       - `get(timeout: float | None = None) -> T`: Acquires lock, `while len == 0` waits on `not_empty`, pops item, calls `not_full.notify()`.
     - `reproduce_abba_deadlock()`: Two threads acquiring `(lock_a, lock_b)` in opposite orders; fix via canonical lock ordering (`id(lock_a) < id(lock_b)`).
   - **Verify:** Stress test with 8 producer threads and 8 consumer threads pushing 100k items; verify zero lost or corrupted items and inspect hung deadlock threads with `py-spy dump --pid <pid>`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 1114 *Print in Order*, LC 1115 *Print FooBar Alternately*, LC 1188 *Design Bounded Blocking Queue* (write from memory).
   - **HLD (30 min):** Contrast Mutex (OS sleep/context switch) vs Spinlock (busy-wait CPU burn—when is spinning faster?), and explain **Priority Inversion** + Priority Inheritance.
   - **LLD (30 min):** Design a `CustomThreadPoolExecutor(num_workers, max_queue_size)` supporting `submit(fn) -> Future` and graceful `shutdown(wait=True)`.

---

### Day 34 — CPython Memory Model: `PyObject`, GC, `pymalloc` & `__slots__` (`3.1`)
1. **Pre-Requisite Learning Resource:**
   - *Fluent Python* (2nd Ed.) — Chapter 6 (*Object References, Mutability, and Recycling*) + CPython Developer's Guide (`devguide.python.org/internals/garbage_collector/` — Reference Counting & Generational Cyclic GC) + `tracemalloc` docs.
2. **Hands-On Activity:**
   - **What to build:** A CPython memory forensics lab (`src/python_lab/mem_internals.py`) measuring object overhead, cyclic GC pauses, and leak patterns in long-running services.
   - **Functions & Components:**
     - `measure_dict_vs_slots(n=500_000)`: Allocates 500k `RegularRow` vs `SlottedRow(__slots__)` instances under `tracemalloc`, reporting peak RSS and MiB saved.
     - `create_reference_cycle_and_inspect()`: Creates `NodeA <-> NodeB`, inspects `sys.getrefcount`, disables cyclic GC (`gc.disable()`), shows RSS leaking across iterations, then runs `gc.collect()` to verify generation 0/1/2 counts.
     - `weakref_cache_demo()`: Uses `weakref.WeakValueDictionary` so cached objects vanish automatically when callers drop references.
   - **Break Intentionally:** Populate an unbounded module-level `dict` cache in a dummy endpoint and track RSS growth across 10k requests using `memray` (`memray run` + `memray flamegraph`).
3. **Daily Split:**
   - **DSA (1 hr):** LC 297 *Serialize and Deserialize Binary Tree* (BFS/preorder null markers).
   - **HLD (30 min):** Why do long-running Python worker processes sometimes grow RSS steadily even without leaks (**allocator arena fragmentation** in `pymalloc`/`ptmalloc`), and why do teams configure `max_requests` / `jemalloc`?
   - **LLD (30 min):** Design an `ObjectPool[T]` (`acquire()`, `release()`) reusing expensive parser/buffer objects with max pool bounds and health validation on return.

---

### Day 35 — CPython Execution Loop, GIL, `threading` vs `multiprocessing` (`3.2`)
1. **Pre-Requisite Learning Resource:**
   - *Fluent Python* (2nd Ed.) — Chapter 19 (*Concurrency Models in Python*: Threads, Processes, GIL, Free-Threaded PEP 703 overview) + Python Docs (`multiprocessing` *Contexts and start methods: `fork` vs `spawn`*).
2. **Hands-On Activity:**
   - **What to build:** A workload-classification benchmark (`src/python_lab/gil_bench.py`) proving when threads win, when processes win, and when `fork` breaks multi-threaded apps.
   - **Functions & Components:**
     - `cpu_heavy_hash(n_iters: int)`: Pure Python SHA-256/math loop.
     - `io_heavy_wait(url_or_sleep)`: Blocking socket/file I/O (releases GIL!).
     - `run_benchmark_matrix()`: Runs both workloads sequentially, with `ThreadPoolExecutor(8)`, and with `ProcessPoolExecutor(4)`, printing wall time and CPU utilization (`htop` / `ps`).
     - `shared_memory_demo()`: Uses `multiprocessing.shared_memory.SharedMemory` to pass a 50 MB buffer between processes without pickle serialization overhead.
   - **Break Intentionally:** Demonstrate `fork()` deadlocking a child process when a parent thread holds a lock at `fork` time vs safe `spawn` / `forkserver`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 200 *Number of Islands* & LC 133 *Clone Graph* (DFS/BFS visited invariants).
   - **HLD (30 min):** Given a data ingestion service doing 80% HTTP download + 20% heavy gzip/Parquet CPU parsing, design the hybrid process+async worker topology.
   - **LLD (30 min):** Design a `HybridExecutor` routing `IO` tasks to a bounded thread/async pool and `CPU` tasks to a process pool with backpressure.

---

### Day 36 — `asyncio` Internals: Coroutines, Event Loop, `TaskGroup` & Cancellation (`3.3`)
1. **Pre-Requisite Learning Resource:**
   - *Fluent Python* (2nd Ed.) — Chapter 21 (*Asynchronous Programming*: coroutines, event loops, `TaskGroup`, async context managers) + Python Docs (*`asyncio` TaskGroups & Cancellation*).
2. **Hands-On Activity:**
   - **What to build:** A structured-concurrency async client aggregator (`src/python_lab/async_internals.py`) with bounded concurrency, timeouts, and cancellation cleanup.
   - **Functions & Components:**
     - `fetch_with_limit(sem: asyncio.Semaphore, client: httpx.AsyncClient, url: str) -> dict`: Acquires semaphore, applies `asyncio.timeout(2.0)`, ensures cleanup inside `try ... except asyncio.CancelledError: ... raise` (never swallow `CancelledError` silently!).
     - `fan_out_requests(urls: list[str], max_concurrency: int)`: Uses `async with asyncio.TaskGroup()` so one fatal failure cancels sibling tasks cleanly.
     - `offload_blocking_parse(raw_bytes: bytes)`: Calls `await asyncio.to_thread(cpu_parse, raw_bytes)`.
   - **Break Intentionally:** Put a synchronous `time.sleep(2)` inside an async route and load-test 20 concurrent requests -> capture the single-core event loop freeze using `py-spy record -o profile.svg`, then fix it.
3. **Daily Split:**
   - **DSA (1 hr):** LC 994 *Rotting Oranges* & LC 417 *Pacific Atlantic Water Flow* (multi-source BFS).
   - **HLD (30 min):** Contrast `uvicorn` async workers (1 process handles thousands of I/O-bound connections via `epoll`/`uvloop`) vs `gunicorn` sync workers (1 process/thread blocked per request).
   - **LLD (30 min):** Design an `AsyncRateLimitedClient` combining `asyncio.Semaphore` (concurrency bound) + async token bucket (QPS bound) + `AsyncExitStack` resource cleanup.

---

### Day 37 — CPU & Memory Profiling (`py-spy`, `memray`, `cProfile`) & WSGI vs ASGI (`3.5`)
1. **Pre-Requisite Learning Resource:**
   - `memray` Docs (*Flamegraphs & Native Mode*) + `py-spy` README (`github.com/benfred/py-spy`) + `uvicorn` Docs (*Deployment & Workers*).
2. **Hands-On Activity:**
   - **What to build:** Create an intentionally buggy FastAPI service (`src/python_lab/slow_service.py`) hiding 3 production bugs: (1) an $O(N^2)$ hot-loop serialization bug, (2) a hidden memory leak per request, and (3) a blocking file read on the event loop.
   - **Functions & Components:**
     - `/bug/cpu`: Quadratic string/list scan inside response building.
     - `/bug/mem`: Appends request payloads into an unbounded cache list.
     - `/bug/block`: Synchronous large `hashlib` / sleep call without `to_thread`.
   - **Inspect & Fix:** Attach `py-spy top --pid <pid>` and `py-spy record` while hitting with `httpx`, plus `memray run` to locate exact lines causing CPU/RAM spikes; compare `gunicorn -k uvicorn.workers.UvicornWorker` vs sync worker throughput.
3. **Daily Split:**
   - **DSA (1 hr):** LC 207 *Course Schedule* & LC 210 *Course Schedule II* (Kahn's BFS Topological Sort + cycle detection).
   - **HLD (30 min):** Draw how `gunicorn` master manages $2 \times \text{cores} + 1$ worker processes and why CPU-bound Python services still hit limits per container.
   - **LLD (30 min):** Design an `AdaptiveWorkerController` monitoring event-loop lag (scheduling a `0.01s` timer and measuring actual wake-up drift!).

---

### Day 38 — Storage Engine Foundations: Slotted Pages, Heap Files & Append-Only Logs (`6.1`)
1. **Pre-Requisite Learning Resource:**
   - *Database Internals* (Alex Petrov) — Chapter 3 (*File Formats*: Slotted Pages, Heap Files, Fixed vs Variable Length Records) + *DDIA* Chapter 3 (*Data Structures That Power Your Database*: Hash Indexes & Log Compaction, pp. 69–76).
2. **Hands-On Activity:**
   - **What to build:** Part 1 of your **Mini Persistent KV Storage Engine** (`src/storage_engine/log_store.py`) implementing Bitcask-style append-only storage with durability and crash recovery.
   - **Functions & Components:**
     - `RecordCodec`: Encodes/decodes binary frames `[crc32 (4B) | timestamp (8B) | key_len (4B) | val_len (4B) | is_tombstone (1B) | key_bytes | val_bytes]` using `struct.pack`.
     - `LogStore`:
       - `_load_index()`: Scans segment file on startup, validates CRC32 checksums (detects partial torn tail writes!), rebuilds `dict[bytes, KeyDirEntry(offset, size)]`.
       - `put(key: bytes, val: bytes)`: Appends binary frame, calls `os.fsync(fd)` (configurable), updates `keydir`.
       - `get(key: bytes) -> bytes | None`: Seeks directly to exact file offset (`os.pread`), reads `size` bytes, verifies CRC, returns value.
       - `delete(key: bytes)`: Writes a tombstone record and removes key from `keydir`.
   - **Break Intentionally:** Append 10 garbage bytes to the tail of the log file (simulating a power loss mid-write) and verify `_load_index()` detects the bad CRC and safely truncates/ignores the corrupt tail.
3. **Daily Split:**
   - **DSA (1 hr):** LC 323 *Number of Connected Components* (or LC 547 *Number of Provinces*) & LC 684 *Redundant Connection* (Union-Find with path compression + union by rank).
   - **HLD (30 min):** Contrast Row-Oriented storage (slotted heap pages for OLTP point lookups/updates) vs Columnar storage (contiguous column pages for OLAP scans).
   - **LLD (30 min):** Design a `SlottedPage` class (`8192` byte bytearray) supporting `insert_tuple(bytes) -> slot_id`, `get_tuple(slot_id)`, and `delete_tuple(slot_id)` with growing header-down / data-up offsets.

---

### Day 39 — Log Compaction & LSM-Tree Internals (Memtable, WAL, SSTables) (`6.2`)
1. **Pre-Requisite Learning Resource:**
   - *DDIA* Chapter 3 (*SSTables and LSM-Trees*, pp. 76–85) + *Database Internals* Chapter 7 (*Log-Structured Storage*: Memtables, SSTables, Compaction, Read/Write/Space Amplification).
2. **Hands-On Activity:**
   - **What to build:** Part 2 of your Mini Storage Engine (`src/storage_engine/lsm_store.py`): transform your append-only log into a real **mini LSM-Tree** with sorted SSTables, sparse index, and compaction.
   - **Functions & Components:**
     - `Memtable`: Sorted in-memory structure (`bisect` or sorted list/dict flushed sorted) with byte-size threshold (`max_bytes`) + backing WAL file for crash recovery.
     - `SSTableWriter.flush(memtable, path)`: Writes sorted KV entries sequentially to immutable `.sst` file + builds an in-memory **sparse index** (every $N$-th key -> byte offset) + optional Bloom filter.
     - `SSTableReader.get(key)`: Binary searches sparse index to find tight `[start_offset, end_offset]` range, scans only that small disk block.
     - `LSMStore.compact()`: Performs a $K$-way merge (using `heapq`!) across immutable SSTables, keeping only the newest version per key and dropping tombstones on full compaction.
   - **Verify:** Write 50k updates to 5k keys -> verify disk size shrinks dramatically after `compact()` and reads still return latest values/tombstones accurately.
3. **Daily Split:**
   - **DSA (1 hr):** LC 23 *Merge k Sorted Lists* (re-implement cleanly thinking of SSTable $K$-way merge!) & LC 743 *Network Delay Time* (Dijkstra).
   - **HLD (30 min):** Compare Size-Tiered Compaction (lower write amplification, high space/read amp—Cassandra) vs Leveled Compaction (bounded space/read amp, higher write amp—RocksDB).
   - **LLD (30 min):** Design `CompactionPlanner` deciding which SSTable levels to merge based on level byte thresholds and overlapping key ranges `[min_key, max_key]`.

---

### Day 40 — B-Tree / B+Tree Internals & Indexing Trade-offs (`6.2`)
1. **Pre-Requisite Learning Resource:**
   - *Database Internals* Chapter 2 (*B-Tree Basics*) & Chapter 4 (*Implementing B-Trees*: Page splits, sibling pointers, fanout) + *Use The Index, Luke* Chapter 1 (*Anatomy of an SQL Index*).
2. **Hands-On Activity:**
   - **What to build:** A pure-Python in-memory **B+Tree** (`src/storage_engine/bplustree.py`) with small order (`order=4` so splits happen fast!) supporting insert, point lookup, and range scan (`[start_key, end_key]`).
   - **Functions & Components:**
     - `BPlusNode`: `is_leaf: bool`, `keys: list`, `children: list` (internal) or `values: list` + `next_leaf` pointer (leaf).
     - `search(key) -> V | None`: Descends root -> internal -> leaf.
     - `range_scan(start_key, end_key) -> list[tuple]`: Finds `start_key` leaf once, then follows `next_leaf` linked pointers!
     - `insert(key, val)`: Splits full leaf/internal nodes and promotes median key upward; creates new root when root splits.
   - **Verify:** Insert 1,000 keys in random order, validate sorted leaf linkage end-to-end, and calculate tree height for branching factor $B=500$ vs $10^9$ rows ($\approx 3$–$4$ hops!).
3. **Daily Split:**
   - **DSA (1 hr):** LC 787 *Cheapest Flights Within K Stops* & LC 1584 *Min Cost to Connect All Points* (Graph shortest path / MST).
   - **HLD (30 min):** Compare B+Tree (Postgres/MySQL InnoDB) vs LSM-Tree (RocksDB/Cassandra) across: random write throughput, point read latency, range scans, write amplification, and tail latency during splits vs compactions.
   - **LLD (30 min):** Design a `BufferPoolManager` (`fetch_page(page_id)`, `unpin_page(page_id, is_dirty)`, `flush_all()`) combining `PagedFileManager` + `LRUCache`.

---

### Day 41 — Transactions, WAL, ARIES Recovery & Isolation Anomalies (`6.3`)
1. **Pre-Requisite Learning Resource:**
   - *DDIA* Chapter 7 (*Transactions*: Weak Isolation Levels, Dirty Reads, Dirty Writes, Read Skew, Lost Updates, Write Skew, Phantoms, Serializable Snapshot Isolation, pp. 221–266) + *Database Internals* Chapter 6 (ARIES WAL overview).
2. **Hands-On Activity:**
   - **What to build:** Two-part lab on Ubuntu VM:
     1. Write an **ARIES-style WAL recovery trace validator** (`src/storage_engine/wal_sim.py`) simulating `BEGIN`, `WRITE(page, old, new)`, `COMMIT`, `ABORT`, `CHECKPOINT` records across a crash.
     2. Stand up **PostgreSQL 16** in Docker (`mem_limit: 1g`) and build the schema for Month 2 DB experiments.
   - **Functions & Components (`wal_sim.py`):**
     - `AnalysisPhase(wal_records)`: Finds active uncommitted transactions and dirty pages at crash time.
     - `RedoPhase(wal_records)`: Replays *all* logged writes forward from last checkpoint LSN (repeating history).
     - `UndoPhase(wal_records, uncommitted_txns)`: Walks backward applying `old_val` and writing Compensation Log Records (CLRs).
   - **Verify:** Simulate crash with 1 committed txn (unflushed dirty page) and 1 uncommitted txn (flushed dirty page—STEAL/NO-FORCE!); verify final pages reflect only the committed txn.
3. **Daily Split:**
   - **DSA (1 hr):** LC 127 *Word Ladder* & LC 269 *Alien Dictionary* (or LC 802 *Find Eventual Safe States* — graph cycle/topo).
   - **HLD (30 min):** Walk through why `STEAL / NO-FORCE` buffer policy requires both Undo and Redo in WAL, and why `fsync` of WAL happens *before* `COMMIT` returns OK while data pages flush lazily.
   - **LLD (30 min):** Design an in-memory `WaitForGraphDeadlockDetector` (`add_wait_edge(tx_a, tx_b)`, `remove_txn(tx)`, `find_cycle() -> list[int] | None`).

---

### Day 42 — PostgreSQL Architecture, `pageinspect`, MVCC & `VACUUM` (`7.2`)
1. **Pre-Requisite Learning Resource:**
   - *The Internals of PostgreSQL* (`interdb.jp/pg/`) — Chapter 1 (*Database Cluster, Postmaster, Shared Memory*) & Chapter 5 (*Concurrency Control*: `xmin`, `xmax`, `ctid`, Visibility Rules) & Chapter 6 (*VACUUM Processing* & HOT updates).
2. **Hands-On Activity:**
   - **What to build:** A hands-on Postgres 16 MVCC forensic lab on Ubuntu VM using `psql` + `pageinspect` extension (`sql/mvcc_lab.sql`).
   - **Functions & Components:**
     - Create table `accounts(id PK, balance INT, note TEXT)` with `CREATE EXTENSION pageinspect;`.
     - Insert rows, run `UPDATE` 5 times on row 1, run `DELETE` on row 2; query `SELECT xmin, xmax, ctid, * FROM accounts;` AND `SELECT lp, t_xmin, t_xmax, t_ctid, t_infomask FROM heap_page_items(get_raw_page('accounts', 0));`.
     - Measure table bloat (`pg_relation_size`), run `VACUUM VERBOSE accounts`, re-inspect dead tuple reuse vs `VACUUM FULL` (`ACCESS EXCLUSIVE` lock warning!).
     - Inspect `TOAST` table creation when inserting a 20 KB string (`EXTENDED` compression vs `EXTERNAL`).
   - **Verify:** Observe `txid_current()` advancing and prove how a long-running idle transaction in Session B prevents `VACUUM` in Session A from reclaiming dead tuples!
3. **Daily Split:**
   - **DSA (1 hr):** LC 70 *Climbing Stairs*, LC 198 *House Robber*, LC 213 *House Robber II* (1D DP state transitions).
   - **HLD (30 min):** Why does Postgres suffer **table bloat** and **transaction ID (`xid`) wraparound** under high-update workloads (unlike MySQL InnoDB's undo log), and what operational alerts catch `autovacuum` falling behind?
   - **LLD (30 min):** Design an in-memory `MVCCVersionChainStore` (`begin_tx() -> tx_id`, `read(key, snapshot_tx_id)`, `write(key, val, tx_id)`, `commit(tx_id)`) implementing snapshot visibility rules.

---

### Day 43 — PostgreSQL Isolation Anomalies & Locks Hands-On (`6.3`, `7.4`)
1. **Pre-Requisite Learning Resource:**
   - PostgreSQL 16 Official Docs — Chapter 13 (*Concurrency Control*: §13.2 Transaction Isolation, §13.3 Explicit Locking) + *The Internals of PostgreSQL* Chapter 5.7 (SSI).
2. **Hands-On Activity:**
   - **What to build:** Execute and script side-by-side two-session `psql` anomaly reproductions (`src/db_lab/isolation_repro.py` using two independent `psycopg` connections).
   - **Functions & Components:**
     - `repro_lost_update_read_committed()`: Two concurrent threads read `balance=100`, add `50`, write back -> final balance is `150` instead of `200`! Fix via `SELECT ... FOR UPDATE` and atomic `UPDATE ... SET balance = balance + 50`.
     - `repro_write_skew_repeatable_read()`: On-call doctors table invariant ("at least 1 doctor must remain on call"). Two concurrent `REPEATABLE READ` txns both count `2`, each updates a *different* row to `off_call` -> invariant violated (`0` doctors)!
     - `verify_ssi_catches_write_skew()`: Switch both txns to `ISOLATION LEVEL SERIALIZABLE` -> one succeeds, one fails with SQLSTATE `40001` (`serialization_failure`) + retry loop fixes it.
     - `repro_deadlock()`: Two threads lock row 1 and row 2 in opposite order -> inspect `40P01 deadlock detected` and `pg_locks`.
   - **Verify:** Automated `pytest` proving the bug under `REPEATABLE READ` and clean retry convergence under `SERIALIZABLE`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 322 *Coin Change* & LC 300 *Longest Increasing Subsequence* (DP + $O(n \log n)$ patience sorting).
   - **HLD (30 min):** Compare Pessimistic Locking (`SELECT FOR UPDATE`) vs Optimistic Concurrency Control (`version` column / `If-Match`) vs `SERIALIZABLE` (SSI) for a ticket-booking seat reservation service.
   - **LLD (30 min):** Design a `@with_serializable_retry(max_retries=5)` database transaction decorator catching SQLSTATE `40001` and `40P01` with jittered backoff.

---

### Day 44 — PostgreSQL Query Lifecycle, CBO Statistics & `EXPLAIN (ANALYZE, BUFFERS)` (`7.3`)
1. **Pre-Requisite Learning Resource:**
   - PostgreSQL 16 Official Docs — Chapter 14.1 (*Using `EXPLAIN`*) & Chapter 14.2 (*Statistics Used by the Planner*) + *Use The Index, Luke* Chapter 2–3 (*The `WHERE` Clause & Performance*).
2. **Hands-On Activity:**
   - **What to build:** A 2-million-row Postgres benchmark & query-plan lab (`src/db_lab/explain_lab.py`) inspecting every scan and join node.
   - **Functions & Components:**
     - `seed_2m_orders()`: Bulk-loads `users(100k)` and `orders(2M)` via Postgres `COPY` (measure `COPY` speed vs row-by-row `INSERT`!).
     - Queries to run with `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)`:
       1. Unindexed `WHERE user_id = 42` (`Seq Scan` + `Buffers: shared read=...`) vs indexed (`Index Scan` + `shared hit=4`).
       2. Covering index `CREATE INDEX ... ON orders(user_id) INCLUDE (total_cents)` -> prove `Index Only Scan` + `Heap Fetches: 0` (after `VACUUM` updates Visibility Map!).
       3. Skewed status column (`99% 'completed'`, `1% 'pending'`) -> inspect `pg_stats` (`most_common_vals`, `most_common_freqs`, `correlation`) and watch Postgres pick `Seq Scan` for `'completed'` vs `Index Scan` for `'pending'`.
       4. Partial index `WHERE status = 'pending'` -> measure 100x smaller index size.
   - **Break Intentionally:** Run a query right after bulk load *without* `ANALYZE` -> observe estimated rows=`1` vs actual rows=`500,000` causing Postgres to pick Nested Loop instead of Hash Join!
3. **Daily Split:**
   - **DSA (1 hr):** LC 139 *Word Break* & LC 518 *Coin Change II*.
   - **HLD (30 min):** Explain how a Cost-Based Optimizer (CBO) uses histograms, MCVs, and index correlation (`pg_stats`) to choose between Sequential Scan, Bitmap Heap Scan, and Index Scan.
   - **LLD (30 min):** Design a `QueryPlanCostEstimator` model computing estimated I/O + CPU cost for `SeqScan(pages)` vs `IndexScan(selectivity, tree_height, correlation)`.

---

### Day 45 — Postgres Join Algorithms, Advanced Index Types (`GIN`, `GiST`, `BRIN`) & Advanced SQL (`7.1`, `7.3`)
1. **Pre-Requisite Learning Resource:**
   - *The Internals of PostgreSQL* — Chapter 3 (*Query Processing*: Nested Loop, Hash Join, Merge Join) + PostgreSQL 16 Docs — Chapter 11 (*Index Types*: `GIN`, `GiST`, `BRIN`, Expression/Partial Indexes) & Chapter 7.8 (`WITH` Queries / Recursive CTEs).
2. **Hands-On Activity:**
   - **What to build:** A Postgres join-spill & specialized-index benchmark (`sql/joins_and_indexes.sql`).
   - **Functions & Components:**
     - **Join Comparison:** Toggle `SET enable_hashjoin = off;` / `enable_mergejoin = off;` with `EXPLAIN (ANALYZE, BUFFERS)` joining `users` and `orders`. Shrink `SET work_mem = '1MB';` during a big Hash Join/Sort to force `Batches: 8` / `Disk` spill (`temp read/written` buffers!).
     - **Specialized Indexes:** Test `GIN` index on `JSONB` metadata (`@>` containment), `BRIN` index on append-only `created_at` (compare size vs B-Tree—tiny KB vs tens of MB!), and expression index on `LOWER(email)`.
     - **Advanced SQL Drills (`7.1`):** Write (1) an org-hierarchy **Recursive CTE**, (2) a per-user top-3 orders query using `CROSS JOIN LATERAL`, (3) `ROLLUP` + `FILTER (WHERE ...)`, and (4) `INSERT ... ON CONFLICT DO UPDATE` (`UPSERT`) + `IS DISTINCT FROM` null-safe diff.
   - **Break Intentionally:** Write `WHERE id NOT IN (SELECT parent_id FROM categories)` where one `parent_id` is `NULL` -> observe `0` rows returned and fix with `NOT EXISTS`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 5 *Longest Palindromic Substring* & LC 647 *Palindromic Substrings*.
   - **HLD (30 min):** When would you pick `BRIN` over `B-Tree` on a 500 GB time-series table, and why does `work_mem` set too high cause Postgres OOM under 200 concurrent connections?
   - **LLD (30 min):** Design an in-memory `HashJoinOperator` (`build_hash_table(small_rel)`, `probe(large_rel)`) and `SortMergeJoinOperator` with memory spill threshold.

---

### Day 46 — Zero-Downtime Postgres DDL, `PgBouncer`, Replication & Partitioning (`7.4`)
1. **Pre-Requisite Learning Resource:**
   - GoCardless / Braintree Engineering Blog — *Safe Zero-Downtime PostgreSQL Migrations* (Lock Queue problem) + PgBouncer Docs (*Pooling Modes*: Session vs Transaction vs Statement) + PostgreSQL 16 Docs (§5.11 Table Partitioning, §27.2 Streaming Replication).
2. **Hands-On Activity:**
   - **What to build:** Multi-container Postgres Primary + Streaming Standby + `PgBouncer` stack on Ubuntu VM (`mem_limit` capped) + Declarative Partitioning lab.
   - **Functions & Components:**
     - `docker-compose.yml`: `pg_primary`, `pg_replica` (streaming WAL replication slot), `pgbouncer` (`pool_mode = transaction`, `max_client_conn = 200`, `default_pool_size = 10`).
     - **Partitioning (`7.4`):** Create `events` partitioned by `RANGE (created_at)` into monthly partitions; run `EXPLAIN ANALYZE` proving **Partition Pruning** skips irrelevant partitions.
     - **Zero-Downtime DDL Checklist:** Practice `CREATE INDEX CONCURRENTLY`, adding a nullable column, and adding a `NOT VALID` check constraint + `VALIDATE CONSTRAINT` (`SHARE UPDATE EXCLUSIVE` instead of `ACCESS EXCLUSIVE`).
   - **Break Intentionally:**
     1. Connect via `PgBouncer` in `transaction` mode using Python `psycopg` with prepared statements enabled -> hit the prepared statement collision error and fix it.
     2. Simulate lock-queue pileup (`ALTER TABLE` waiting behind a long read) and enforce `SET lock_timeout = '2s'` retry script.
3. **Daily Split:**
   - **DSA (1 hr):** LC 72 *Edit Distance* & LC 1143 *Longest Common Subsequence* (2D DP).
   - **HLD (30 min):** Explain why Postgres hits connection overhead around 300–500 active processes (process-per-connection memory + lock contention) and how `PgBouncer` + Read Replicas scale read/connection capacity.
   - **LLD (30 min):** Design a `ZeroDowntimeMigrationRunner` executing DDL statements with `lock_timeout`, `statement_timeout`, and jittered retry on lock contention.

---

### Day 47 — Caching Architectures: Strategies, Eviction & Stampede Prevention (`5.1`)
1. **Pre-Requisite Learning Resource:**
   - AWS Caching Best Practices (*Caching Patterns*: Cache-Aside, Read-Through, Write-Through, Write-Behind) + Go `singleflight` / Facebook Memcache Paper (*Scaling Memcache at Facebook* §2–3: leases, thundering herd, cold-cluster warmup).
2. **Hands-On Activity:**
   - **What to build:** Upgrade your FastAPI + Postgres URL Shortener with **Redis 7** caching (`src/cache/redis_cache.py`) implementing Cache-Aside, TTL Jitter, Null-Caching, and **Singleflight Request Coalescing**.
   - **Functions & Components:**
     - `SingleflightGroup`: Tracks `in_flight: dict[str, asyncio.Future]` keyed by cache key; if 100 concurrent coroutines miss Redis for the same key simultaneously, only 1 executes the Postgres query while 99 `await` the shared `Future`.
     - `CacheAsideURLRepo(URLRepository)`: Wraps `PostgresURLRepository`; `get_by_code` checks Redis -> on miss runs inside `singleflight.do(code, ...)` -> caches result with `base_ttl + random.uniform(0, jitter)` -> caches explicit `__NULL__` sentinel for short TTL (`30s`) to prevent **cache penetration** on bogus keys; `save`/`update` invalidates key.
   - **Break Intentionally:** Load-test 100 concurrent requests to a cold hot-key with `Singleflight` OFF (count Postgres queries via `pg_stat_statements` -> 100 DB queries!) vs ON (1 DB query!).
3. **Daily Split:**
   - **DSA (1 hr):** LC 460 *LFU Cache* (HashMap + frequency-keyed Doubly Linked Lists in $O(1)$).
   - **HLD (30 min):** Compare Cache-Aside vs Write-Through vs Write-Behind (data-loss risk on crash!) and solve Hot-Key skew (e.g., celebrity profile hitting 1 Redis shard at 200k RPS -> L1 in-memory cache + key salting `key#{0..9}`).
   - **LLD (30 min):** Design a `WTinyLFUCache` conceptually / `SegmentedLRUCache` with admission filter (`CountMinSketch` frequency check before evicting protected segment).

---

### Day 48 — Redis Internals, Persistence & Cluster Topology (`5.2`)
1. **Pre-Requisite Learning Resource:**
   - Redis Official Docs (*Redis Persistence: RDB vs AOF*, *Redis Cluster Specification* §Hash Slots & Redirections, *Redis Pipelining*) + Redis source inspection (`src/sds.c` & `src/t_zset.c` comments/overview).
2. **Hands-On Activity:**
   - **What to build:** A Redis internals & latency-benchmark lab (`src/cache/redis_internals.py`) on Ubuntu VM inspecting encodings, pipelining, persistence, and big-key stalls.
   - **Functions & Components:**
     - `inspect_encodings()`: Inserts small vs large strings, hashes, and sorted sets; queries `OBJECT ENCODING` (`int`, `embstr`, `raw`, `listpack`, `hashtable`, `skiplist`).
     - `benchmark_pipelining(n=10_000)`: Compares 10,000 sequential `SET` round-trips vs 100-command `pipeline()` batches.
     - `test_big_key_stall()`: Inserts a 1M-element list/hash, runs `DEL bigkey` (blocks single-threaded Redis event loop!) vs `UNLINK bigkey` (async background free); monitors `LATENCY DOCTOR` / `SLOWLOG GET`.
     - Configure Redis `RDB` + `AOF` (`appendfsync everysec`) and `maxmemory 256mb` + `allkeys-lru` eviction.
   - **Verify:** Trigger `BGSAVE` and read `INFO persistence` (`rdb_last_cow_size`, `latest_fork_usec`) to see `fork()` Copy-on-Write memory risk.
3. **Daily Split:**
   - **DSA (1 hr):** LC 1206 *Design Skiplist* (probabilistic multi-level linked list—literally how Redis `ZSET` works!).
   - **HLD (30 min):** Explain Redis Cluster's 16,384 hash slots (`CRC16(key) % 16384`), hash tags `{user:123}:profile` for multi-key operations, and `MOVED` vs `ASK` redirections during resharding.
   - **LLD (30 min):** Design an `InMemorySkipList` supporting `search`, `insert`, and `range_by_score(min_s, max_s)`.

---

### Day 49 — Atomic Redis Lua Scripting & Distributed Rate Limiters (`5.2`, `13.1`)
1. **Pre-Requisite Learning Resource:**
   - Redis Docs (*Programmability: Eval Scripts & Atomicity*, *Pattern: Rate Limiter*) + Figma / Stripe Rate-Limiting engineering posts.
2. **Hands-On Activity:**
   - **What to build:** A distributed **Sliding-Window Counter** & **Token Bucket** rate limiter implemented via **atomic Redis Lua scripts** (`src/ratelimit/redis_lua.py`), wired as FastAPI middleware.
   - **Functions & Components:**
     - `TOKEN_BUCKET_LUA`: Executes atomically inside Redis's single thread (`EVALSHA`): reads `tokens`, `last_ts` from Hash `KEYS[1]`, refills based on `redis.call('TIME')` (never trust client app clocks!), deducts cost if allowed, sets `EXPIRE`, returns `[allowed, remaining, retry_after_ms]`.
     - `SLIDING_WINDOW_LUA`: Tracks current & previous fixed-window counters and computes weighted overlap count.
     - `RateLimitMiddleware`: Extracts API key / IP, enforces limits, attaches `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `429 Too Many Requests` + `Retry-After`; implements configurable **fail-open** if Redis times out (`>20ms`).
   - **Break Intentionally:** Stop Redis container mid-load and prove your API degrades gracefully (`fail-open` with warning metric) instead of throwing `500` on every request!
3. **Daily Split:**
   - **DSA (1 hr):** LC 362 *Design Hit Counter* & LC 359 *Logger Rate Limiter*.
   - **HLD (30 min):** Design a **Distributed Rate Limiter** handling 500k RPS globally (compare Token Bucket vs Sliding Window Log vs Sliding Window Counter, local token batching vs Redis Cluster, fail-open vs fail-closed).
   - **LLD (30 min):** Design `RedisScriptLoader` (`register_script`, `evalsha_with_noscript_fallback`) handling `NOSCRIPT` reload automatically on Redis restart.

---

### Day 50 — Task Queues: Postgres `SKIP LOCKED` vs Broker Queues (`5.3`)
1. **Pre-Requisite Learning Resource:**
   - Brandur Leach (`brandur.org/postgres-queues`) — *Queueing in Postgres* + *2ndQuadrant / PG Docs* on `FOR UPDATE SKIP LOCKED`.
2. **Hands-On Activity:**
   - **What to build:** A production-grade Postgres-backed Background Job Queue (`src/jobs/pg_queue.py`) with workers, visibility timeouts, retries, and a Dead-Letter Queue (DLQ).
   - **Functions & Components:**
     - `jobs` table: `id`, `tenant_id`, `queue_name`, `payload JSONB`, `status` (`pending`, `running`, `done`, `dead`), `attempts`, `max_attempts`, `run_at`, `locked_until`, `last_error`.
     - `enqueue(tenant_id, payload, delay_sec=0)`: Inserts pending job.
     - `claim_batch(worker_id, batch_size=5, visibility_sec=30) -> list[Job]`: Atomically selects `WHERE status = 'pending' AND run_at <= now() ORDER BY run_at FOR UPDATE SKIP LOCKED LIMIT :batch_size` and updates `status='running', locked_until = now() + 30s` in **one CTE `UPDATE ... RETURNING`**.
     - `ack_job(job_id)` / `nack_job(job_id, error)`: On `nack`, increments `attempts`; if `< max_attempts`, computes exponential backoff + jitter for `run_at`, else moves to `status='dead'` (DLQ).
     - `reclaim_stale_jobs()`: Periodically resets `running` jobs whose `locked_until < now()` (handles crashed workers!).
   - **Break Intentionally:** Run 4 concurrent worker processes, kill `-9` one worker mid-job, and prove `reclaim_stale_jobs()` recovers the orphaned job after `visibility_sec` without losing or blocking other workers.
3. **Daily Split:**
   - **DSA (1 hr):** LC 78 *Subsets* & LC 46 *Permutations* & LC 39 *Combination Sum* (backtracking recursion tree).
   - **HLD (30 min):** Design a Multi-Tenant Background Job Platform preventing one noisy customer dumping 1M jobs from starving other customers (**Weighted Fair Queuing / per-tenant queue sharding**).
   - **LLD (30 min):** Design a `FairTenantScheduler` (`enqueue(tenant_id, job)`, `dequeue_next()`) using deficit round-robin or per-tenant queues.

---

### Day 51 — Idempotency Keys & Outbound Webhook Delivery System (`4.3`, `4.4`)
1. **Pre-Requisite Learning Resource:**
   - Stripe Engineering Blog — *Designing robust and predictable APIs with idempotency* (`stripe.com/blog/idempotency`) + Standard Webhooks Spec (`standardwebhooks.com`).
2. **Hands-On Activity:**
   - **What to build:** Add Stripe-style `Idempotency-Key` protection on `POST /v1/orders` AND an `HMAC-SHA256` signed outbound Webhook Delivery Worker (`src/webhooks/delivery.py`).
   - **Functions & Components:**
     - `IdempotencyRecord` table (`key PK`, `request_hash`, `status` (`in_progress`/`completed`), `response_code`, `response_body`, `created_at`).
     - `IdempotencyMiddleware`: Inserts `(key, hash, 'in_progress') ON CONFLICT DO NOTHING`:
       - If conflict & `completed` & hash matches -> replay exact saved response (`200`/`201`) without re-executing handler!
       - If conflict & hash differs -> `422 Idempotency Key Reuse Mismatch`.
       - If conflict & `in_progress` -> `409 Conflict` (`Retry-After: 1`)!
     - `WebhookSigner`: Builds `signature = hmac_sha256(secret, f"{timestamp}.{body}")`, sends `X-Webhook-Timestamp` + `X-Webhook-Signature`; receiver verifies signature (`hmac.compare_digest`) AND rejects timestamps older than 5 minutes (anti-replay!).
     - `WebhookDispatcher`: Uses Day 50's job queue + per-subscriber **Circuit Breaker** so one dead subscriber doesn't stall other webhooks.
   - **Verify:** Fire 10 simultaneous identical `POST /v1/orders` with the same `Idempotency-Key` -> verify exactly 1 order row created in Postgres.
3. **Daily Split:**
   - **DSA (1 hr):** LC 79 *Word Search* & LC 131 *Palindrome Partitioning*.
   - **HLD (30 min):** Design a **Webhook Delivery Platform** (10k tenants, slow/down endpoints, HMAC verification, retry backoff, per-endpoint circuit breaking, DLQ replay UI).
   - **LLD (30 min):** Design `WebhookVerifier` (`verify(headers, raw_body, secret, tolerance_sec=300)`) handling secret rotation (current + previous secret overlap window).

---

### Day 52 — Full-Text Search & Elasticsearch / Lucene Internals (`5.4`)
1. **Pre-Requisite Learning Resource:**
   - Elasticsearch Official Docs (*Inverted Index*, *Analysis & Analyzers*, *Search After Pagination*, *Lucene Segments & Refresh Interval*) + *BM25* formula intuitive overview.
2. **Hands-On Activity:**
   - **What to build:** Run single-node Elasticsearch/OpenSearch on Ubuntu VM (`ES_JAVA_OPTS="-Xms1g -Xmx1g"`) alongside a search indexer script (`src/search/es_lab.py`) and compare with Postgres `tsvector`.
   - **Functions & Components:**
     - `create_index_with_custom_analyzer()`: Configures char filter (`html_strip`), standard tokenizer, lowercase + stop + snowball stemmer + edge-ngram autocomplete filter; inspects output via `_analyze`.
     - `bulk_index_documents(docs)`: Indexes 100k docs, observes `refresh_interval=1s` near-real-time visibility vs segment creation (`_cat/segments`).
     - `search_bm25(query, explain=True)`: Inspects TF-IDF / BM25 scoring breakdown (`explain: true`), field boosts (`title^3`), and `search_after` + Point-In-Time (`PIT`) deep pagination.
   - **Break Intentionally:** Run deep pagination `from=50000&size=20` (hits `index.max_result_window=10000` error + heap pressure) and replace with `search_after` + `PIT`; also execute zero-downtime reindex behind an **index alias**.
3. **Daily Split:**
   - **DSA (1 hr):** LC 212 *Word Search II* (Trie + DFS backtracking pruning).
   - **HLD (30 min):** Design **Typeahead / Search Autocomplete** + Full-Text Product Search (inverted index sharding, scatter-gather query phase vs fetch phase, why `from/size` explodes memory on coordinators).
   - **LLD (30 min):** Build a mini `InvertedIndex` class in Python (`index_doc(doc_id, text)`, `tokenize()`, `search_bm25(query)`) computing real TF, IDF, and length normalization!

---

### Day 53 — Caching Eviction Policies, Bloom Penetration Guard & L1/L2 (`5.1`, `13.1`)
1. **Pre-Requisite Learning Resource:**
   - Gil Einziger et al. — *TinyLFU: A Highly Efficient Cache Admission Policy* (§1–3) + *Caffeine* cache design blog (`github.com/ben-manes/caffeine/wiki/Design`).
2. **Hands-On Activity:**
   - **What to build:** Add a **Two-Tier Cache (L1 In-Process LRU + L2 Redis)** + **Bloom Filter Penetration Guard** (`src/cache/two_tier.py`) in front of Postgres.
   - **Functions & Components:**
     - `SimpleBloomFilter(expected_items, fp_rate)`: Uses `bytearray` bitset and double hashing (`h1 + i*h2`) to reject nonexistent short codes *before* hitting Redis or Postgres!
     - `TwoTierCache`: Checks L1 (`LRUCache`, tiny 5s TTL for hot keys) -> on miss checks Bloom filter (if `False`, immediately returns `404`!) -> checks L2 (Redis) -> on miss hits Postgres inside `Singleflight`.
     - **Invalidation Pub/Sub:** When a URL is updated/deleted, publish key invalidation on Redis Pub/Sub so all API instances purge L1 immediately (`5.1`).
   - **Verify:** Attack the API with 50,000 random nonexistent `GET /v1/urls/bogus123` requests -> confirm **0 Postgres queries** executed!
3. **Daily Split:**
   - **DSA (1 hr):** LC 17 *Letter Combinations of a Phone Number* & LC 51 *N-Queens* (state pruning).
   - **HLD (30 min):** How do you mitigate **Cache Penetration** (random missing IDs), **Cache Avalanche** (all keys expire at midnight simultaneously), and **Hot-Key Failure** (single viral key overloading 1 Redis shard)?
   - **LLD (30 min):** Design a `BloomFilter` + `CountingBloomFilter` (supporting deletion via 4-bit counters—explain overflow trade-off).

---

### Day 54 — Integration Testing With Ephemeral `Testcontainers` & Property Testing (`16.1`)
1. **Pre-Requisite Learning Resource:**
   - `testcontainers-python` Official Docs (*Postgres & Redis modules*) + `Hypothesis` Docs (*Property-Based Testing Quickstart*).
2. **Hands-On Activity:**
   - **What to build:** Upgrade your test suite (`tests/integration/`) to spin up real ephemeral Postgres + Redis containers and write **property-based tests** (`Hypothesis`) for your Base62 encoder, LRU cache, and Token Bucket.
   - **Functions & Components:**
     - `conftest.py`: Session-scoped `PostgresContainer` & `RedisContainer` fixtures running schema migrations once and truncating tables per test.
     - `test_url_shortener_end_to_end()`: Verifies `POST /v1/urls` -> Postgres persistence -> Redis population -> `GET` hit -> `If-Match` `412` conflict.
     - `test_base62_roundtrip_property(@given(st.integers(min_value=0, max_value=2**63)))`: Fuzzes thousands of random edge integers.
     - `test_lru_model_based(@rule)`: Stateful Hypothesis test comparing your custom `LRUCache` against a trivial `OrderedDict` reference model across random `get`/`put` sequences!
   - **Verify:** Run the full integration + property suite on Ubuntu VM.
3. **Daily Split:**
   - **DSA (1 hr):** LC 416 *Partition Equal Subset Sum* & LC 494 *Target Sum*.
   - **HLD (30 min):** Design a fast, deterministic CI testing pyramid for a backend service (Unit fakes -> `Testcontainers` Postgres/Redis/Kafka -> OpenAPI contract checks).
   - **LLD (30 min):** Design a `StatefulModelTestHarness` verifying invariants across arbitrary sequences of `deposit`/`withdraw`/`transfer` operations.

---

### Day 55 — Postgres + Redis + Queue Mini-Capstone: Payment/Order API (`4.3`, `5.1`, `5.3`, `7.4`)
1. **Pre-Requisite Learning Resource:**
   - Review your Month 2 notes + PostgreSQL Docs on `SELECT ... FOR UPDATE` and `READ COMMITTED` vs `REPEATABLE READ`.
2. **Hands-On Activity:**
   - **What to build:** Integrate everything in Month 2 into one rock-solid **Order & Wallet Service** on Ubuntu VM (`src/orders_app/`):
     - FastAPI (`asyncio`) + `PgBouncer` -> Postgres 16 (partitioned `ledger_entries` + `orders` + `jobs` tables).
     - `Idempotency-Key` middleware + atomic wallet debit (`CHECK (balance >= 0)` + `FOR UPDATE` or `SERIALIZABLE` retry).
     - Enqueues async confirmation webhook in `jobs` (`SKIP LOCKED`) inside the **same DB transaction** as the order creation!
     - Redis `Lua` rate limiter + `Singleflight` product-catalog cache + OpenAPI contract test + `Testcontainers` integration tests.
   - **Verify:** Fire concurrent duplicate checkouts, concurrent overdraft attempts, and kill a webhook worker mid-flight—verify zero negative balances, zero lost webhooks, and zero duplicate charges.
3. **Daily Split:**
   - **DSA (1 hr):** LC 53 *Maximum Subarray* (Kadane's) & LC 152 *Maximum Product Subarray*.
   - **HLD (30 min):** Draw the complete Order & Wallet architecture, tracing synchronous ACID write path vs asynchronous webhook fan-out path.
   - **LLD (30 min):** Design `TransferService.transfer(from_acct, to_acct, amount, idempotency_key)` preventing deadlocks when `A->B` and `B->A` run concurrently (hint: canonical lock order `min(id), max(id)`!).

---

### Days 56–60 — Month 2 Deep-Dive Consolidation, Profiling & Drills

#### Day 56 — Profiling & Fixing Month 2 Capstone Under Load (`3.5`, `7.3`, `17.3`)
1. **Pre-Requisite Learning Resource:**
   - PostgreSQL Docs (*`pg_stat_statements`*) + `py-spy` README.
2. **Hands-On Activity:**
   - **What to build:** Stress-test Day 55's Order & Wallet API with 200 concurrent clients on Ubuntu VM while capturing `pg_stat_statements`, `EXPLAIN (ANALYZE, BUFFERS)`, `py-spy` flamegraph, and `docker stats`.
   - **Functions & Components:**
     - Enable `pg_stat_statements` and write a diagnostic query ranking top-5 queries by `total_exec_time` and `shared_blks_read`.
     - Tune connection pool size, add missing composite/covering indexes, eliminate any blocking calls in `asyncio`, and record before/after `P95`/`P99` throughput.
3. **Daily Split:**
   - **DSA (1 hr):** LC 55 *Jump Game* & LC 45 *Jump Game II* & LC 134 *Gas Station* (greedy invariants).
   - **HLD (30 min):** How do you systematically isolate whether a `P99` spike is caused by Python CPU/GIL, blocked `asyncio` loop, `PgBouncer` pool exhaustion, Postgres lock waits, or disk I/O?
   - **LLD (30 min):** Design a `SlowQueryAuditor` middleware tagging requests exceeding 50ms DB time with normalized query fingerprints.

#### Day 57 — Storage Engine & Postgres Internals Recall Sprint (`6.1–6.3`, `7.2`)
1. **Pre-Requisite Learning Resource:**
   - Review *DDIA* Ch. 3 & Ch. 7 summary diagrams + *The Internals of PostgreSQL* Ch. 5–6.
2. **Hands-On Activity:**
   - **What to build:** Clean-room drill on Ubuntu VM: extend your Day 39 **mini LSM-Tree** to support range scans (`scan(start_key, end_key)` merging memtable + multiple SSTables via min-heap while shadowing deleted/older keys!).
   - **Functions & Components:** `range_scan(start, end) -> Iterator[tuple[bytes, bytes]]` using a `(key, -seq_num, sstable_id, val)` heap to deduplicate keys cleanly across levels.
   - **Verify:** Run property tests (`Hypothesis`) comparing LSM `range_scan` against a plain sorted Python `dict` across random puts, overwrites, and deletes!
3. **Daily Split:**
   - **DSA (1 hr):** LC 62 *Unique Paths* & LC 64 *Minimum Path Sum* & LC 309 *Best Time to Buy and Sell Stock with Cooldown*.
   - **HLD (30 min):** Explain every step of a write and read inside RocksDB vs PostgreSQL (WAL -> Memtable/SharedBuffers -> SSTable/HeapPage -> Compaction/VACUUM).
   - **LLD (30 min):** Design `WriteAheadLog` (`append(record)`, `sync()`, `replay() -> Iterator`, `truncate_before(lsn)`).

#### Day 58 — Redis & Queue Failure Simulation Day (`5.1–5.3`)
1. **Pre-Requisite Learning Resource:**
   - Redis Docs (*Redis Latency Monitoring Framework* & *Memory Optimization*).
2. **Hands-On Activity:**
   - **What to build:** Execute 4 live failure simulations on Ubuntu VM:
     1. Trigger Redis `OOM` (`maxmemory 32mb` + `noeviction`) -> observe write errors vs `allkeys-lru` eviction behavior.
     2. Kill Redis unexpectedly and verify `Singleflight` + `RateLimiter` fail-open without crashing FastAPI.
     3. Inject poison-pill malformed JSON into `jobs` table -> verify retry backoff moves it to `dead` (DLQ) without blocking healthy jobs.
     4. Simulate 2 workers racing for the same job without `SKIP LOCKED` vs with `SKIP LOCKED` -> measure throughput difference.
3. **Daily Split:**
   - **DSA (1 hr):** LC 202 *Happy Number* & LC 50 *Pow(x, n)* & LC 48 *Rotate Image* (math/matrix).
   - **HLD (30 min):** Design **Instagram / Social Media Feed** (`fanout-on-write` to Redis lists vs `fanout-on-read` for 50M-follower celebrities, hybrid merge).
   - **LLD (30 min):** Design a `CircuitBreaker` state machine with sliding failure-rate window (`deque` of recent outcomes) instead of naive consecutive-failure count.

#### Day 59 — Elasticsearch & Search Architecture Drill (`5.4`)
1. **Pre-Requisite Learning Resource:**
   - Elasticsearch Docs (*Shard Allocation*, *Document Routing*, *Aliases & Zero-Downtime Reindexing*).
2. **Hands-On Activity:**
   - **What to build:** Build a zero-downtime reindexing pipeline script (`src/search/reindex_lab.py`) on Ubuntu VM:
     - `create_v1_index()` + alias `products_read` -> `products_v1`.
     - `create_v2_index()` (updated mapping/analyzer) -> `_reindex` from `products_v1` to `products_v2` -> atomic alias swap (`remove products_v1`, `add products_v2`) -> verify zero search downtime during concurrent queries!
   - **Verify:** Stop Elasticsearch afterward (`docker compose down`) to reclaim Ubuntu VM RAM for Month 3!
3. **Daily Split:**
   - **DSA (1 hr):** LC 1268 *Search Suggestions System* (Trie / Binary Search prefix matching).
   - **HLD (30 min):** Design **Uber Eats Restaurant Search & Ranking** (Geo-bounding box filter + Inverted Index text match + BM25/ML re-ranking + Redis cache).
   - **LLD (30 min):** Design a `QueryAliasRouter` supporting dual-write migration (`write_both`, `shadow_read_diff`, `cutover`) during search schema upgrades.

#### Day 60 — Month 2 Comprehensive Assessment
1. **Pre-Requisite Learning Resource:**
   - Audit [Curriculum.md](./Curriculum.md) Sections `1.2–1.4`, `3.1–3.3`, `3.5`, `4.4`, `5.1–5.4`, `6.1–6.3`, `7.1–7.4`.
2. **Hands-On Activity:**
   - **What to build:** Clean-room timed diagnostic (2 hrs max, zero AI):
     1. Given a slow Postgres query on 1M rows (prepared deliberately), diagnose via `EXPLAIN (ANALYZE, BUFFERS)` and drop latency >50x.
     2. Attach `py-spy` to a running Python worker and identify which function blocks the `asyncio` event loop.
     3. Verify `docker compose down` leaves Ubuntu VM clean with >14 GB free RAM ready for Month 3's distributed clusters.
3. **Daily Split:**
   - **DSA (1 hr):** Timed 60-min Mock: LC 460 *LFU Cache* + LC 210 *Course Schedule II*.
   - **HLD (30 min):** Timed 30-min Blank-Page Design: **Ticketmaster / Flash-Sale Inventory Booking System** (hot-key seat locking, Redis holds + Postgres `FOR UPDATE` / `SERIALIZABLE` checkout, queue waiting room).
   - **LLD (30 min):** Timed 30-min Blank-Page Design: **Thread-Safe `LRUCache` with per-entry TTL and background reaper**.
