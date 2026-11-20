# Month 4 (Days 91–120): Debezium CDC, Apache Flink, Apache Spark Internals, Lakehouse & OLAP Engines

> **Curriculum Coverage:** `10.4`, `11.1–11.3`, `12.1–12.4`, `14.3`, `16.1` (data pipeline testing)
> **Environment:** Main Laptop + Ubuntu 16 GB VM (Run ONE heavy stack at a time: Days 91–103 = Postgres + Kafka + Debezium + Flink; Days 104–120 = Spark + MinIO/Iceberg + DuckDB + ClickHouse)

---

### Day 91 — Change Data Capture (CDC): Polling vs Triggers vs Log-Based Debezium (`10.4`)
1. **Pre-Requisite Learning Resource:**
   - Debezium Docs (*Architecture* + *PostgreSQL Connector: Logical Decoding & `pgoutput`*) + Gunnar Morling — *Incremental Snapshots in Debezium (DBLog Watermark Algorithm)* (`debezium.io/blog/2021/10/07/incremental-snapshots/`).
2. **Hands-On Activity:**
   - **What to build:** Stand up **Postgres (`wal_level=logical`) + 1-broker KRaft Kafka + Kafka Connect + Debezium Postgres Connector** on Ubuntu VM (cap Connect JVM at `768m`).
   - **Functions & Components:**
     - `register_debezium_connector()`: POSTs JSON config (`plugin.name: pgoutput`, `publication.autocreate.mode: filtered`, `slot.name: debezium_orders`, `tombstones.on.delete: true`).
     - `cdc_inspector.py`: Consumes `dbserver1.public.orders` and parses `op` (`r` = snapshot read, `c` = insert, `u` = update with `before`/`after`, `d` = delete + tombstone `value=None`).
   - **Verify:** Run `INSERT`, `UPDATE`, `DELETE` in Postgres `psql` and watch `cdc_inspector.py` print exact before/after row images within milliseconds.
3. **Daily Split:**
   - **DSA (1 hr):** LC 295 (re-solve Two Heaps) & LC 480 *Sliding Window Median* (two heaps + lazy deletion map).
   - **HLD (30 min):** Contrast Query Polling (`WHERE updated_at > ?` — misses deletes & bursts!), DB Triggers (write-latency tax), and Log-Based CDC (`pgoutput` reading WAL asynchronously).
   - **LLD (30 min):** Design an `IdempotentCDCMerger` applying out-of-order `(op, lsn_or_ts, before, after)` events into a downstream materialized table without regressing newer rows.

---

### Day 92 — CDC Failure Modes: Replication Slots, Heartbeats & Schema Evolution (`10.4`)
1. **Pre-Requisite Learning Resource:**
   - Debezium Docs (*PostgreSQL Connector: WAL Disk Space Consumption & Heartbeats* + *Schema Change Events*).
2. **Hands-On Activity:**
   - **What to build:** Reproduce and fix the #1 production CDC outage—**idle replication slot WAL disk exhaustion**—and test DDL schema evolution (`src/cdc_lab/slot_guard.py`).
   - **Functions & Components:**
     - `slot_monitor()`: Polls `pg_replication_slots` computing `pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)` in MB (`active` boolean + retained WAL bytes).
     - Enable `heartbeat.interval.ms` + `heartbeat.action.query` updating a tiny `debezium_heartbeat` table so `confirmed_flush_lsn` advances even when monitored tables sit quiet while *other* DB tables write heavily!
   - **Break Intentionally:** Pause/stop the Debezium connector, generate 100k updates in Postgres, and watch retained WAL MB climb in `pg_replication_slots` until you restart the connector or set `max_slot_wal_keep_size`! Also `ALTER TABLE orders ADD COLUMN coupon TEXT` and inspect the new CDC payload.
3. **Daily Split:**
   - **DSA (1 hr):** LC 315 *Count of Smaller Numbers After Self* (Merge Sort inversion counting / Fenwick Tree).
   - **HLD (30 min):** How does the **DBLog Watermark Algorithm** take an initial consistent snapshot of a 1 TB table *without* holding long locks (`REPEATABLE READ` table lock) while live streaming changes continue?
   - **LLD (30 min):** Design a `WalRetentionCircuitBreaker` alerting and safely dropping/archiving dead replication slots when retained WAL exceeds `80%` disk threshold.

---

### Day 93 — Stream Processing Time Semantics: Event Time, Watermarks & Windows (`11.1`)
1. **Pre-Requisite Learning Resource:**
   - Tyler Akidau — *The World Beyond Batch: Streaming 101* & *Streaming 102* (`oreilly.com/radar/the-world-beyond-batch-streaming-101/` — Event time vs Processing time, Watermarks, Windows, Triggers).
2. **Hands-On Activity:**
   - **What to build:** Before opening Flink, **build a mini pure-Python Event-Time Window & Watermark Engine** (`src/stream_sim/watermark_engine.py`) so streaming stops being a myth!
   - **Functions & Components:**
     - `StreamEvent(key, event_ts_ms, value)`.
     - `BoundedOutOfOrdernessWatermarkGenerator(max_out_of_order_ms=5000)`: Tracks `max_seen_event_ts`; emits `watermark = max_seen_event_ts - max_out_of_order_ms`.
     - `TumblingWindowOperator(window_size_ms=10_000, allowed_lateness_ms=3_000)`:
       - Assigns event to `[start, end)` window bucket.
       - Fires window output once `watermark >= window.end`.
       - If a late event arrives with `window.end <= watermark < window.end + allowed_lateness_ms`, updates and emits a **late-firing retraction/update**!
       - If `watermark >= window.end + allowed_lateness_ms`, drops state and routes event to a **side output** (`late_events`).
   - **Verify:** Feed shuffled out-of-order events (`t=12s, 14s, 8s, 22s, 2s`) and trace every watermark advance, window firing, and side-output capture.
3. **Daily Split:**
   - **DSA (1 hr):** LC 1229 / LC 986 *Interval List Intersections* & LC 759 *Employee Free Time* (sweep line / min-heap).
   - **HLD (30 min):** Explain the difference between **Tumbling**, **Sliding/Hopping**, and **Session** windows (inactivity gap merging!) and how **multi-partition watermark alignment** handles 1 idle Kafka partition blocking the global watermark.
   - **LLD (30 min):** Design a `SessionWindowMerger` merging overlapping `[start, end + gap]` session intervals per `user_id` as out-of-order clicks arrive.

---

### Day 94 — Apache Flink Setup, DataStream API & Windowing (`11.1`, `11.3`)
1. **Pre-Requisite Learning Resource:**
   - Apache Flink Official Docs — *Concepts: Stateful Stream Processing*, *Timely Stream Processing (Event Time & Watermarks)*, *Windows*, and *Flink Architecture (JobManager, TaskManager, Slots)*.
2. **Hands-On Activity:**
   - **What to build:** Spin up **Apache Flink** (1 JobManager + 1 TaskManager, `taskmanager.memory.process.size: 1536m`) alongside Kafka on Ubuntu VM; build a **PyFlink / Flink SQL** windowed aggregation job (`src/flink_lab/window_job.py`).
   - **Functions & Components:**
     - Event generator producing JSON click/order events to Kafka topic `events.raw` with intentional 0–15s random event-time jitter.
     - Flink job: Defines `WatermarkStrategy.for_bounded_out_of_orderness(Duration.of_seconds(5))` + `with_idleness(Duration.of_seconds(10))` (crucial for idle partitions!), computes 1-minute tumbling and 1-minute-slide-10s hopping counts per `city_id`.
     - Routes events exceeding `allowed_lateness` to a Kafka `events.late` side-output topic.
   - **Verify:** Inspect Flink Web UI (`:8081`) operator DAG, **Watermarks** tab, and busy/backpressure metrics!
3. **Daily Split:**
   - **DSA (1 hr):** LC 239 (re-solve Sliding Window Max) & LC 1438 *Longest Continuous Subarray With Absolute Diff <= Limit* (two monotonic deques!).
   - **HLD (30 min):** Contrast **Continuous Operator Streaming (Flink)** (record-by-record pipelined operators, sub-second latency, credit-based flow control) vs **Micro-Batch Streaming (Spark Structured Streaming)** (Driver schedules mini-batch tasks every trigger interval).
   - **LLD (30 min):** Design a `MultiPartitionWatermarkTracker` computing `min(partition_watermarks)` while marking quiet partitions `IDLE` after timeout so they don't stall global progress.

---

### Day 95 — Stateful Flink: Keyed State, RocksDB & Chandy-Lamport Checkpoints (`11.2`)
1. **Pre-Requisite Learning Resource:**
   - Carbone et al. — *Lightweight Asynchronous Snapshots for Distributed Dataflows* (Flink's Chandy-Lamport barriers, §1–4) + Flink Docs (*Working with State*, *State Backends: HashMap vs RocksDB*, *Checkpoints vs Savepoints*).
2. **Hands-On Activity:**
   - **What to build:** Add **Keyed State + RocksDB Incremental Checkpointing** (`src/flink_lab/stateful_fraud.py`) implementing a real-time fraud rule ("alert if a `user_id` spends >$500 across 3+ distinct merchants within 5 minutes") + State TTL.
   - **Functions & Components:**
     - `FraudDetector(KeyedProcessFunction)`: Uses `ValueState` (running total), `MapState` (merchant -> timestamp), `StateTtlConfig` (auto-expires keys after 10 min so state doesn't grow unbounded!), and `on_timer()` event-time cleanup.
     - Configure `EmbeddedRocksDBStateBackend(enable_incremental=True)` writing checkpoints every 10s to local disk / MinIO S3.
   - **Break Intentionally:** While streaming 200 events/sec, **`docker kill` the Flink TaskManager**, wait 5s, restart it (`docker start`), and verify Flink restores RocksDB state + Kafka offsets from the last completed checkpoint with **zero lost alerts and zero inflated counts**!
3. **Daily Split:**
   - **DSA (1 hr):** LC 381 *Insert Delete GetRandom O(1) - Duplicates allowed* & LC 1472 *Design Browser History*.
   - **HLD (30 min):** Trace how **Chandy-Lamport Checkpoint Barriers** flow through a streaming DAG: what happens during an **Aligned Checkpoint** (fast channel pauses waiting for slow channel barrier -> backpressure!) vs an **Unaligned Checkpoint** (in-flight buffers stored inside snapshot).
   - **LLD (30 min):** Design a `KeyedStateBackendWithTTL` storing `(value, last_access_ts)` and lazily evicting expired entries on read/compaction.

---

### Day 96 — Stream-Table Duality, Temporal Joins & Exactly-Once Sinks (`11.2`)
1. **Pre-Requisite Learning Resource:**
   - Jay Kreps / Martin Kleppmann — *DDIA* Chapter 11 (*Stream Processing*: Streams and Tables, Enriching Streams, Stream-Table Joins, pp. 441–475) + Flink Docs (*Temporal Joins & Two-Phase Commit Sink*).
2. **Hands-On Activity:**
   - **What to build:** Build an end-to-end **Debezium CDC + Flink Stream-Table Enrichment** pipeline (`src/flink_lab/cdc_enrich.py`): join live `orders` stream against a CDC `exchange_rates` changelog table (`KTable`) using an event-time **Temporal Join** (point-in-time currency conversion!).
   - **Functions & Components:**
     - Source 1: Kafka `orders` stream (event time `order_ts`).
     - Source 2: Debezium CDC `exchange_rates` compacted changelog stream (versioned table with `valid_from` / `rowtime`).
     - Temporal Join: Matches each order with the exact exchange rate active at `order_ts` (NOT today's rate—crucial for deterministic replay!).
     - Sink: Idempotent upsert (`INSERT ... ON CONFLICT DO UPDATE`) into Postgres.
   - **Verify:** Update the FX rate mid-stream and replay historical orders -> prove historical orders still compute using the historical rate!
3. **Daily Split:**
   - **DSA (1 hr):** LC 715 *Range Module* (or LC 732 *My Calendar III*) — interval tree / sorted boundaries.
   - **HLD (30 min):** Compare **Idempotent Upsert Sinks** (cheap, works for Postgres/ClickHouse/Redis) vs **Two-Phase Commit (2PC) Sinks** (`FlinkKafkaProducer` pre-commit on checkpoint barrier -> commit on checkpoint complete).
   - **LLD (30 min):** Design a `VersionedTemporalLookupTable` keeping `(key, valid_ts) -> value` versions in memory and answering `get_as_of(key, event_ts)` while pruning versions older than `watermark`.

---

### Day 97 — Streaming Backpressure, Skew Mitigation & Kafka Streams (`11.3`)
1. **Pre-Requisite Learning Resource:**
   - Apache Flink Blog — *How Flink Handles Backpressure* (Credit-Based Flow Control) + Confluent Kafka Streams Docs (*Architecture: Tasks, State Stores, Standby Replicas*).
2. **Hands-On Activity:**
   - **What to build:** Reproduce **stream backpressure** and **key-skew bottlenecks** in Flink on Ubuntu VM (`src/flink_lab/skew_backpressure.py`) and fix both.
   - **Functions & Components:**
     - **Backpressure Test:** Add an artificial `20ms` slow sink -> watch Flink Web UI show Sink task **Busy 100%** and upstream Map/Source tasks **Backpressured 100%** (buffers fill upstream -> Kafka consumer pauses pulling!).
     - **Skewed Stream Fix:** Feed 90% of events for `merchant_id="amazon"` into a `key_by(merchant_id)` aggregation -> 1 TaskManager slot melts while 3 sit idle. Implement **Two-Phase Local-Global Aggregation**: Phase 1 keys by `(merchant_id, rand(0..7))` (pre-aggregates across 8 slots!), Phase 2 keys by `merchant_id` (sums only the 8 partial results!).
   - **Verify:** Tear down the Flink + Debezium + Kafka Docker Compose stack afterward (`docker compose down`) to free all 16 GB RAM for Spark!
3. **Daily Split:**
   - **DSA (1 hr):** LC 1094 *Car Pooling* & LC 1109 *Corporate Flight Bookings* (difference array prefix sums).
   - **HLD (30 min):** Compare when a team should pick **Kafka Streams** (embedded library inside Python/JVM microservice, RocksDB + changelog topic + standby replicas, no separate cluster) vs **Apache Flink** (dedicated compute cluster, complex event-time SQL/CEP/huge state) vs **Spark Structured Streaming**.
   - **LLD (30 min):** Design a `TwoPhaseSaltedAggregator` (`pre_aggregate(key, salt_buckets, val)`, `merge_partials(key, partial_vals)`).

---

### Day 98 — Apache Spark Architecture: Driver, Executors, Lineage & Catalyst (`12.1`)
1. **Pre-Requisite Learning Resource:**
   - Apache Spark Docs (*Cluster Mode Overview*, *RDD Programming Guide: Transformations vs Actions*) + Databricks Blog (*Deep Dive into Spark SQL's Catalyst Optimizer*) + *Spark: The Definitive Guide* Ch. 15 (*How Spark Runs on a Cluster*).
2. **Hands-On Activity:**
   - **What to build:** Install PySpark locally on your Ubuntu 16 GB VM (`local[4]`, `spark.driver.memory=2g`, `spark.executor.memory=2g`) and build a **Spark UI & Physical Plan Inspector** (`src/spark_lab/catalyst_dag.py`).
   - **Functions & Components:**
     - Generate a 5M-row Parquet dataset (`events` + `users` + `countries`).
     - Run 3 queries and inspect `df.explain("extended")` + **Spark UI (`localhost:4040`) Jobs / Stages / SQL tabs**:
       1. Filter + column projection + arithmetic -> observe **1 Stage, 0 Shuffle**, Predicate Pushdown (`PushedFilters`), and Column Pruning (`ReadSchema`) in `Physical Plan`!
       2. `groupBy("country").count()` -> observe **2 Stages** separated by `Exchange hashpartitioning` (Partial HashAggregate -> Shuffle -> Final HashAggregate!).
       3. Inspect **Whole-Stage Code Generation (`*(1)`, `*(2)`)** and `UnsafeRow` binary execution (`Tungsten`).
   - **Verify:** Trace exactly why `count()` triggers a Spark Job while `.filter().withColumn()` builds lazy lineage with zero execution.
3. **Daily Split:**
   - **DSA (1 hr):** LC 621 (re-solve) & LC 1353 *Maximum Number of Events That Can Be Attended* (Greedy + Min-Heap).
   - **HLD (30 min):** Walk through what happens when a Spark Executor dies mid-Stage 2: how does Spark recompute lost shuffle blocks using **RDD Lineage** without checkpointing every intermediate step?
   - **LLD (30 min):** Design a `LazyDataFramePlanner` building an AST (`Scan -> Filter -> Project -> Aggregate`), running **Predicate Pushdown** (moving `Filter` below `Project`/`Join` onto `Scan`) before emitting a physical plan!

---

### Day 99 — Spark Shuffle Internals & Join Strategies (`12.2`)
1. **Pre-Requisite Learning Resource:**
   - Apache Spark Docs (*Performance Tuning: Join Strategy Hints*, *Shuffle Behavior*) + *Spark: The Definitive Guide* Ch. 18 (*Monitoring & Debugging*) & Ch. 19 (*Performance Tuning: Shuffle & Joins*).
2. **Hands-On Activity:**
   - **What to build:** Benchmark all **3 Spark Join Algorithms** (`src/spark_lab/joins_bench.py`) while inspecting `Shuffle Read/Write` bytes in Spark UI.
   - **Functions & Components:**
     - `run_broadcast_hash_join()`: Join 5M `events` with 50k `users` (`broadcast(users)`) -> verify `BroadcastHashJoin` + `BroadcastExchange` (**0 shuffle on the 5M table!**).
     - `run_sort_merge_join()`: Disable auto-broadcast (`spark.sql.autoBroadcastJoinThreshold=-1`), join 5M `events` with 1M `orders` -> observe `Exchange` (shuffle both tables!) -> `Sort` -> `SortMergeJoin`.
     - `run_shuffle_hash_join()`: Force `hint("shuffle_hash")` -> observe `ShuffledHashJoin` (no sort, builds hash table per partition—faster if one shuffled side fits in memory, OOM risk if skewed!).
     - Inspect `SortShuffleManager` temp index/data files and `Spill (Memory)` / `Spill (Disk)` by lowering `spark.memory.fraction`.
   - **Break Intentionally:** Try `broadcast()` on a 1.5 GB table with a 1 GB driver/executor -> trigger `BroadcastTimeout` / Driver OOM and explain why the Driver collects the broadcast table first!
3. **Daily Split:**
   - **DSA (1 hr):** LC 347 & LC 692 *Top K Frequent Words* (Heap custom comparator / trie).
   - **HLD (30 min):** Draw the physical network & disk movement for Broadcast Hash Join vs Sort-Merge Join vs **Bucketed Pre-Partitioned Join** (zero shuffle for two huge tables joined repeatedly on the same key!).
   - **LLD (30 min):** Design an `ExternalSortShuffleWriter` sorting `(partition_id, key)` pairs into spill files on disk and merging them into one sorted data file + offset index file.

---

### Day 100 — Spark Data Skew, Salting, AQE & Memory/OOM Diagnosis (`12.1`, `12.2`)
1. **Pre-Requisite Learning Resource:**
   - Databricks Docs / Blog (*Adaptive Query Execution (AQE): Dynamically Coalescing Shuffle Partitions, Switching Join Strategies, Optimizing Skew Joins*) + Apache Spark Docs (*Memory Management Overview*: Execution vs Storage Memory, PySpark Arrow).
2. **Hands-On Activity:**
   - **What to build:** A **Spark Skew & AQE Rescue Lab** (`src/spark_lab/skew_aqe.py`) on Ubuntu VM.
   - **Functions & Components:**
     - `generate_skewed_dataset()`: 5M rows where `key="HOT_CORP"` has 4.2M rows (84%!) and 10k other keys share 800k rows.
     - **Step 1 (Broken Baseline):** Disable AQE (`spark.sql.adaptive.enabled=false`), join skewed dataset -> inspect Spark UI **Tasks** tab sorted by duration: 199 tasks finish in 0.3s, **1 task runs 30x longer** (straggler)!
     - **Step 2 (Manual Key Salting):** Salt `HOT_CORP` into `HOT_CORP_{0..15}` on the big table and explode `0..15` copies of `HOT_CORP` on the small dimension table -> verify balanced task durations!
     - **Step 3 (AQE Auto-Fix):** Enable AQE (`spark.sql.adaptive.enabled=true`, `skewJoin.enabled=true`, `coalescePartitions.enabled=true`) -> inspect Spark UI SQL tab showing `AQEShuffleRead` splitting the skewed partition and coalescing 200 tiny partitions down!
   - **Verify:** Map the 3 distinct Spark OOM signatures: **Driver OOM** (`collect()` / huge broadcast), **Executor JVM OOM** (skew / large `ShuffledHashJoin`), **Python Worker OOM** (heavy UDF / Arrow batch size).
3. **Daily Split:**
   - **DSA (1 hr):** LC 1642 *Furthest Building You Can Reach* & LC 871 *Minimum Number of Refueling Stops* (Lazy Greedy Min-Heap).
   - **HLD (30 min):** Explain **Spark Unified Memory Management** (`reserved 300MB`, `spark.memory.fraction=0.6`, dynamic boundary where Execution can evict cached Storage blocks, and when spill triggers) + **`coalesce()` vs `repartition()`** (why `coalesce` avoids shuffle by merging locally, and why `coalesce(1)` bottlenecks the entire upstream stage onto 1 task!).
   - **LLD (30 min):** Design an `AdaptivePartitionCoalescer` scanning post-shuffle partition byte sizes `[1MB, 2MB, 1MB, 128MB, ...]` and merging adjacent small partitions up to `target_size=64MB`.

---

### Day 101 — Columnar File Internals: Apache Parquet & ORC (`12.3`)
1. **Pre-Requisite Learning Resource:**
   - Apache Parquet Format Spec (`parquet.apache.org/docs/file-format/`) + Julien Le Dem / Netflix / ClickHouse Parquet Internals posts (Row Groups, Column Chunks, Pages, Dictionary/RLE/Bit-Packing, Footer Min/Max Stats, Bloom Filters).
2. **Hands-On Activity:**
   - **What to build:** A **Parquet Binary Anatomy Inspector** (`src/lakehouse_lab/parquet_inspect.py`) using `pyarrow.parquet` on Ubuntu VM.
   - **Functions & Components:**
     - `write_comparison_files()`: Writes the same 2M rows with (a) random unsorted `user_id`, (b) sorted `user_id`, and (c) low-cardinality `country` (`row_group_size=100_000`, `use_dictionary=True`, `write_statistics=True`).
     - `inspect_parquet_footer(path)`: Reads `pq.ParquetFile(path).metadata` to print: number of Row Groups, per-Column Chunk encodings (`RLE_DICTIONARY`, `PLAIN`), compressed vs uncompressed byte sizes, and exact `min`/`max` stats per Row Group!
     - `benchmark_predicate_pushdown()`: Queries `WHERE user_id = 42` via `pyarrow.dataset` on unsorted vs sorted Parquet -> prove sorted Parquet skips 19/20 Row Groups by reading **only the footer** (`min`/`max` pruning)!
   - **Verify:** Compare disk size of CSV vs JSON vs Snappy-Parquet vs Zstd-Parquet on the exact same 2M rows.
3. **Daily Split:**
   - **DSA (1 hr):** LC 443 *String Compression* & LC 394 *Decode String* & Run-Length / Delta encoding logic.
   - **HLD (30 min):** Why do large Parquet Row Groups (128 MB–512 MB) hurt point-filtering latency, while tiny Row Groups (1 MB) ruin compression and bloat footer metadata?
   - **LLD (30 min):** Design a `ColumnarPageEncoder` implementing **Dictionary Encoding + Run-Length Encoding (RLE)** on a column of strings and measuring compression ratio.

---

### Day 102 — Data Lake Failure Modes & Apache Iceberg Internals (`12.3`)
1. **Pre-Requisite Learning Resource:**
   - Apache Iceberg Official Spec (*Table Format Spec*: Snapshots, Manifest Lists, Manifest Files, Hidden Partitioning, Optimistic Concurrency) + Netflix TechBlog (*Introducing Iceberg* — Hive table failure modes).
2. **Hands-On Activity:**
   - **What to build:** Stand up **MinIO + Iceberg REST Catalog + Spark / PyIceberg** on Ubuntu VM (`src/lakehouse_lab/iceberg_inspect.py`) and inspect every metadata file Iceberg creates!
   - **Functions & Components:**
     - Create an Iceberg table `events` with **Hidden Partitioning** (`days(event_ts)` + `bucket(16, user_id)`).
     - Commit 3 writes (`APPEND`, `DELETE`/`MERGE`, `APPEND`) -> open the actual `metadata/v*.metadata.json`, `snap-*.avro` (Manifest List), and `*-m0.avro` (Manifest Files storing per-Parquet-file column `lower_bounds`/`upper_bounds`!) directly in Python!
     - Run **Time Travel** (`VERSION AS OF` / `TIMESTAMP AS OF`) and **Partition Evolution** (`ALTER TABLE ... ADD PARTITION FIELD hours(event_ts)`) without rewriting old files!
   - **Verify:** Contrast Hive's `MSCK REPAIR` / slow S3 `LIST` directory scans against Iceberg's atomic catalog pointer swap (`metadata_location` CAS!).
3. **Daily Split:**
   - **DSA (1 hr):** LC 1472 & LC 1146 *Snapshot Array* (binary search on snapshot versions).
   - **HLD (30 min):** Explain the 5 failure modes of legacy Hive-style Parquet data lakes (non-atomic overwrites, expensive object-store `LIST`, schema drift corruption, partition-column leaks in queries, no concurrent writers) and how Iceberg solves all 5.
   - **LLD (30 min):** Design an `OptimisticCatalogCommitter` (`commit_snapshot(table, expected_metadata_loc, new_metadata_loc)`) retrying on concurrent conflict (`ConcurrentModificationException`) when new snapshot appends don't conflict with deleted files!

---

### Day 103 — Delta Lake, Apache Hudi & Lakehouse Maintenance (`12.3`)
1. **Pre-Requisite Learning Resource:**
   - Armbrust et al. — *Delta Lake: High-Performance ACID Table Storage over Cloud Object Stores* (VLDB '20, §2–4: `_delta_log`, checkpoints, deletion vectors) + Apache Hudi Docs (*Table Types: Copy-on-Write vs Merge-on-Read*).
2. **Hands-On Activity:**
   - **What to build:** Run a **Delta Lake** vs **Iceberg** maintenance lab (`src/lakehouse_lab/maintenance_lab.py`) on Ubuntu VM inspecting `_delta_log` JSON commits, Deletion Vectors / Merge-on-Read deletes, Z-Ordering/Clustering, and Compaction.
   - **Functions & Components:**
     - Write 200 tiny appends to a Delta/Iceberg table -> inspect `000000.json`..`000010.json` + `.checkpoint.parquet` in `_delta_log/`.
     - Perform a 1-row `DELETE` / `UPDATE` (observe CoW rewriting an entire 100 MB file vs MoR / Deletion Vector writing a tiny delta/bitmap!).
     - Run **Compaction (`rewrite_data_files` / `OPTIMIZE`)**, **Z-Order / Clustering** on `(country, user_id)`, **Snapshot Expiration**, and **Orphan File Removal** (never delete orphan files newer than retention window—concurrent writer hazard!).
   - **Verify:** Measure query scan time before vs after Z-Ordering + bin-packing compaction!
3. **Daily Split:**
   - **DSA (1 hr):** LC 54 *Spiral Matrix* & LC 240 *Search a 2D Matrix II* + Z-order curve bit-interleaving concept!
   - **HLD (30 min):** When would a streaming CDC ingestion pipeline pick **Hudi Merge-on-Read (MoR)** or **Iceberg Copy-on-Write + Deletion Vectors** over pure Copy-on-Write?
   - **LLD (30 min):** Design a `BinPackCompactor` selecting small files `< 32 MB` inside each partition and grouping them into `~256 MB` target rewrite jobs while tracking orphan-safe retention timestamps.

---

### Day 104 — MPP Query Engines (Trino/Presto) & Embedded Vectorized OLAP (DuckDB) (`12.4`)
1. **Pre-Requisite Learning Resource:**
   - Trino Docs (*Concepts: Coordinator, Workers, Stages, Tasks, Splits, Dynamic Filtering*) + DuckDB Official Docs / Paper (*Push-Based Vectorized Execution & Morsel-Driven Parallelism*).
2. **Hands-On Activity:**
   - **What to build:** Benchmark **DuckDB** (embedded vectorized C++ OLAP) querying Iceberg/Parquet + Apache Arrow zero-copy against **PySpark** on the exact same 10M-row dataset on Ubuntu VM (`src/olap_lab/duckdb_vs_spark.py`).
   - **Functions & Components:**
     - `duckdb_parquet_query()`: Queries partitioned Parquet/Iceberg directly using `EXPLAIN ANALYZE` in DuckDB, observing vectorized pipeline operators and pushdown.
     - `zero_copy_arrow_exchange()`: Passes a 5M-row Arrow Table between DuckDB -> PyArrow -> Pandas/Polars without copying memory buffers (`pa.total_allocated_bytes()`).
     - **Trino Concepts (`[T2]`/`[T3]`):** Map how Trino's Coordinator plans splits across Workers, uses **Dynamic Filtering** (pushing broadcast join hash-table min/max into table scan splits!), and when Trino spills to disk vs fails OOM.
   - **Verify:** Compare wall-clock runtime and RAM usage of Spark (`local[4]`) vs DuckDB on single-node 10M-row analytical queries.
3. **Daily Split:**
   - **DSA (1 hr):** LC 209 *Minimum Size Subarray Sum* & LC 560 *Subarray Sum Equals K* (prefix sum + hashmap).
   - **HLD (30 min):** When should a Data Platform run **DuckDB** (single-node, <100 GB working set, CI/serverless/interactive notebooks) vs **Trino** (federated multi-TB queries across Iceberg/Hive/Postgres) vs **Spark** (heavy multi-TB ETL/ML shuffles with fault-tolerant task retries)?
   - **LLD (30 min):** Design a `VectorizedColumnBatchOperator` processing `2048`-element typed arrays per step with selection vectors (masking filtered rows without copying arrays!).

---

### Day 105 — Real-Time OLAP: ClickHouse (`MergeTree`) & Apache Pinot (`12.4`)
1. **Pre-Requisite Learning Resource:**
   - ClickHouse Official Docs (*MergeTree Engine Family*, *Primary Keys & Sparse Indexes*, *Materialized Views*) + Apache Pinot Docs (*Architecture & Star-Tree Index* `[T3]`).
2. **Hands-On Activity:**
   - **What to build:** Stand up single-node **ClickHouse** in Docker on Ubuntu VM (`mem_limit: 2g`) and load the same 10M-row events table (`src/olap_lab/clickhouse_lab.py`), comparing **Postgres vs DuckDB vs ClickHouse**.
   - **Functions & Components:**
     - Create `events_ch` using `ENGINE = MergeTree() ORDER BY (tenant_id, event_type, toStartOfHour(ts), user_id)` (no primary-key uniqueness—sorted sparse index every 8,192 rows!).
     - Inspect `system.parts` (`marks`, `rows`, `bytes_on_disk`) as ClickHouse creates immutable parts on batch inserts and merges them in the background.
     - Create an incremental **`MATERIALIZED VIEW` (`SummingMergeTree` / `AggregatingMergeTree`)** pre-aggregating hourly tenant metrics automatically on insert!
   - **Break Intentionally:** Insert 1,000 individual 1-row `INSERT`s synchronously into ClickHouse -> observe `Too many parts` error/degradation and learn why ClickHouse requires batched inserts (or `async_insert=1`).
3. **Daily Split:**
   - **DSA (1 hr):** LC 304 *Range Sum Query 2D - Immutable* & LC 307 *Range Sum Query - Mutable* (Segment Tree / Fenwick Tree).
   - **HLD (30 min):** Why can ClickHouse aggregate 100M rows in <100ms while Postgres takes seconds? Contrast ClickHouse (`MergeTree` + vectorized SIMD + incremental MVs) vs **Apache Pinot** (real-time Kafka ingestion + Star-Tree pre-aggregated cubes for sub-50ms user-facing Uber Eats analytics) vs **Query Federation**.
   - **LLD (30 min):** Design a `SparseGranuleIndex` storing every 8,192nd sorted tuple's `(primary_key_tuple, file_offset)` and finding candidate granules for `WHERE tenant_id = X AND ts >= Y`.

---

### Days 106–112 — Data Governance, Data Quality, Lakehouse Pipeline & Spark Mastery Drills

#### Day 106 — Metadata Catalogs, Data Lineage & Quality Circuit Breakers (`14.3`, `16.1`)
1. **Pre-Requisite Learning Resource:**
   - OpenLineage Spec (`openlineage.io/docs/spec/object-model`) + Netflix / Uber Data Quality Engineering Blogs (*Write-Audit-Publish (WAP) pattern with Iceberg Branches!*).
2. **Hands-On Activity:**
   - **What to build:** Implement the **Write-Audit-Publish (WAP)** pattern on Iceberg/MinIO (`src/lakehouse_lab/wap_pipeline.py`) + deterministic pipeline tests (`16.1`).
   - **Functions & Components:**
     - `write_to_staging_branch()`: Writes daily Spark/DuckDB batch into an Iceberg staging branch (`audit_branch`) invisible to production queries.
     - `run_quality_circuit_breaker()`: Asserts (1) non-empty row count within `[-20%, +50%]` of 7-day rolling median, (2) zero `NULL` primary keys, (3) zero duplicate `(order_id)`, (4) valid schema contract.
     - `publish_branch()`: Fast-forwards `main` Iceberg branch atomically *only* if all quality assertions pass; otherwise leaves `main` untouched and emits alert + OpenLineage JSON event!
   - **Verify:** Inject duplicate PKs + 50% missing rows -> prove bad data is quarantined on `audit_branch` and **never** exposed to downstream dashboards!
3. **Daily Split:**
   - **DSA (1 hr):** LC 269 / LC 210 + LC 1136 *Parallel Courses* (DAG longest-path critical chain!).
   - **HLD (30 min):** Design a **Company-Wide Data Lineage & Quality Platform** (column-level lineage parsing from SQL/Spark/dbt, OpenLineage events, WAP Iceberg circuit breaking, PII tagging).
   - **LLD (30 min):** Design a declarative `DataContractValidator` (`assert_not_null`, `assert_unique`, `assert_freshness`, `assert_distribution_drift`) returning structured pass/fail audit reports.

#### Day 107 — PII Governance, Tokenization & GDPR Erasure in Immutable Lakes (`14.3`)
1. **Pre-Requisite Learning Resource:**
   - Uber / Airbnb Engineering posts on GDPR Right-to-Erasure in Data Lakes & Kafka (Crypto-shredding vs Iceberg Merge-on-Read deletes + Compaction).
2. **Hands-On Activity:**
   - **What to build:** Implement **Crypto-Shredding** + **Iceberg Row-Level GDPR Erasure** (`src/governance_lab/gdpr_erasure.py`).
   - **Functions & Components:**
     - `UserKeyVault`: Stores per-user AES encryption key `user_id -> dek_bytes`.
     - `encrypt_pii_columns(row)` / `decrypt_pii_columns(row)`: Stores encrypted `email_cipher`, `phone_cipher` inside immutable Kafka/Parquet events!
     - `erase_user_gdpr(user_id)`: (1) Deletes `user_id`'s key from `UserKeyVault` (instantly rendering all historical Kafka logs/backups unreadable!), AND (2) executes Iceberg `DELETE FROM events WHERE user_id = ...` + expires old snapshots so deleted PII isn't retained in Time Travel!
   - **Verify:** Attempt to read historical Parquet rows for deleted `user_id` -> returns `[REDACTED]`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 380 & LC 381 (O(1) randomized hash structures).
   - **HLD (30 min):** Compare **Rewriting Parquet partitions on every GDPR delete** (massive write amplification!) vs **Batching deletes weekly + Crypto-shredding** for immutable Kafka + S3/Iceberg lakes.
   - **LLD (30 min):** Design a `ColumnMaskingPolicyEngine` applying role-based `HASH`, `PARTIAL_MASK`, `NULLIFY`, or `PLAINTEXT` transforms at query boundary.

#### Day 108 — Spark End-to-End Tuning & PySpark Arrow UDF Pitfalls (`12.1`, `12.2`)
1. **Pre-Requisite Learning Resource:**
   - Apache Spark Docs (*PySpark Usage Guide for Pandas with Apache Arrow*) + Databricks Guide (*Diagnosing Executor vs Driver vs Python OOMs*).
2. **Hands-On Activity:**
   - **What to build:** Benchmark 3 PySpark anti-patterns vs fixes (`src/spark_lab/pyspark_perf.py`) on Ubuntu VM while watching `htop` & Spark UI.
   - **Functions & Components:**
     - **Anti-Pattern 1:** Python row-by-row `@udf` (pickles every row between JVM <-> Python process!) vs vectorized **Pandas Arrow UDF (`@pandas_udf`)** vs native Catalyst `pyspark.sql.functions` expression (`WholeStageCodegen`).
     - **Anti-Pattern 2:** `df.collect()` of 2M rows crashing Driver vs writing partitioned Parquet/Iceberg.
     - **Anti-Pattern 3:** `repartition(1)` bottlenecking final write vs right-sized `repartition(N)` / `maxRecordsPerFile`.
   - **Verify:** Record 10x–50x speedup moving from Python `@udf` -> `@pandas_udf` -> native Catalyst SQL functions.
3. **Daily Split:**
   - **DSA (1 hr):** LC 973 & LC 253 (Heap interval/distance drills).
   - **HLD (30 min):** Trace memory flow inside a PySpark Executor: JVM Heap + JVM Off-Heap (`UnsafeRow`) + Python Worker RSS (separate OS process not tracked by JVM `-Xmx` -> why Kubernetes kills containers with `OOMKilled 137`!).
   - **LLD (30 min):** Design a `PartitionSizeEstimator` computing optimal output partition count from input compressed bytes, estimated compression ratio (`~4x`), and target Parquet file size (`256 MB`).

#### Day 109 — Lakehouse + Streaming Unified Architecture (Kappa vs Lambda) (`11.2`, `12.3`, `12.4`)
1. **Pre-Requisite Learning Resource:**
   - Jay Kreps — *Questioning the Lambda Architecture* (`oreilly.com/radar/questioning-the-lambda-architecture/`) + *DDIA* Ch. 11/12 (Batch + Stream unification).
2. **Hands-On Activity:**
   - **What to build:** Connect **DuckDB / Spark** and **ClickHouse** over a single **MinIO Iceberg** dataset (`src/olap_lab/unified_query.py`) to test Lakehouse + Real-Time OLAP serving parity.
   - **Functions & Components:**
     - Batch historical data (`D-30` to `D-1`) stored in Iceberg on MinIO/Azure Blob; live `D-0` streaming aggregates served from ClickHouse (`MergeTree`).
     - Write a reconciliation script (`16.1`) verifying idempotent streaming counts match batch Spark recomputation to the exact cent (`reconcile_batch_vs_stream()`).
   - **Verify:** Clean up Spark/ClickHouse containers cleanly afterward (`docker compose down`) or verify disk space (`df -h`).
3. **Daily Split:**
   - **DSA (1 hr):** LC 23 (re-solve) & LC 1383 *Maximum Performance of a Team* (Sort + Min-Heap).
   - **HLD (30 min):** Compare **Lambda** (dual codebases Spark + Flink -> drift bugs!) vs **Kappa** (single streaming engine replaying Kafka/Iceberg) vs **Modern Lakehouse** (Flink writing Iceberg + ClickHouse/Pinot serving low-latency queries).
   - **LLD (30 min):** Design a `StreamBatchReconciler` diffing hourly watermark aggregates between live ClickHouse tables and recomputed Iceberg snapshots with tolerance thresholds.

#### Day 110 — Spark Structured Streaming vs Flink Deep Comparison (`11.3`, `12.1`)
1. **Pre-Requisite Learning Resource:**
   - Apache Spark Docs (*Structured Streaming Programming Guide*: Micro-batch vs Continuous Processing, Watermarking, State Store).
2. **Hands-On Activity:**
   - **What to build:** Port Day 94's 1-minute event-time windowed aggregation to **Spark Structured Streaming** (`src/spark_lab/structured_stream.py`) reading from files/rate source and writing to local Parquet/memory sink (`outputMode="append"` vs `"update"`).
   - **Functions & Components:**
     - Configure `.withWatermark("event_ts", "5 seconds").groupBy(window("event_ts", "1 minute"), "city_id").count()`, `.option("checkpointLocation", ...)` and observe micro-batch progress JSON (`inputRowsPerSecond`, `processedRowsPerSecond`, `watermark`).
   - **Verify:** Contrast Spark's `checkpointLocation` JSON offsets + HDFS-backed/RocksDB state provider vs Flink's Chandy-Lamport barriers in your notes.
3. **Daily Split:**
   - **DSA (1 hr):** LC 56 & LC 57 & LC 452 *Minimum Number of Arrows to Burst Balloons*.
   - **HLD (30 min):** When is **Spark Structured Streaming** (`10s–60s` micro-batch latency) actually a better operational choice than **Flink** (e.g., team already runs Spark/Iceberg and needs unified batch/stream ETL without 50ms SLAs)?
   - **LLD (30 min):** Design a `MicroBatchTriggerLoop` tracking committed batch IDs, reading new file/Kafka offsets atomically, and guaranteeing idempotent sink commits.

#### Day 111 — Data Platform Whiteboarding & Skew/Failure Triage (`11.1–11.3`, `12.1–12.4`)
1. **Pre-Requisite Learning Resource:**
   - Review your Month 4 `notes.md` + Spark UI & Flink Web UI diagnostics.
2. **Hands-On Activity:**
   - **What to build:** Written & diagnostic **Production Incident Simulation (5 ICMs!)**—write step-by-step root-cause runbooks (`docs/month4_icm_runbooks.md`) based on your Month 4 breakages:
     1. Postgres primary disk hits 95% overnight because Debezium stopped (`pg_replication_slots`).
     2. Spark nightly ETL SLA jumps from 20m to 4h (199/200 tasks done, 1 hanging).
     3. Flink checkpoint size grows linearly from 500 MB to 80 GB over 3 days (missing State TTL / uncompacted RocksDB).
     4. ClickHouse throws `Too many parts (300)` under direct microservice inserts.
     5. Iceberg table queries slow down 10x despite small data volume (50,000 tiny 10 KB files).
3. **Daily Split:**
   - **DSA (1 hr):** Timed Mock: LC 480 *Sliding Window Median* + LC 329 *Longest Increasing Path in a Matrix*.
   - **HLD (30 min):** Timed 30-min Blank-Page Design: **Clickstream Analytics Ingestion & Real-Time Funnel Dashboard** (100k events/sec -> Kafka -> Flink/ClickHouse + Iceberg Lakehouse).
   - **LLD (30 min):** Design a `CompactionScheduler` scanning Iceberg tables and prioritizing partitions with highest small-file-count-to-size ratio.

#### Day 112 — Month 4 Deep Technical Audit & Cleanup (`10.4`, `11.1–11.3`, `12.1–12.4`, `14.3`)
1. **Pre-Requisite Learning Resource:**
   - Audit [Curriculum.md](./Curriculum.md) Sections `10.4`, `11.1–11.3`, `12.1–12.4`, `14.3`.
2. **Hands-On Activity:**
   - **What to build:** Clean-room drill on Ubuntu VM:
     1. Write a 40-line PySpark job joining 2 skewed tables, enable AQE, and explain every node in `explain("extended")` aloud.
     2. Inspect a Parquet file's row-group stats + an Iceberg `metadata.json` snapshot pointer without notes.
     3. Stop all Month 4 containers (`docker compose down && docker system prune -f`) so Ubuntu VM is ready for Month 5 `k3s` + Observability + Temporal!
3. **Daily Split:**
   - **DSA (1 hr):** Timed Mock (unseen): LC 1353 *Maximum Number of Events* + LC 743 *Network Delay Time*.
   - **HLD (30 min):** Timed 30-min Blank-Page Design: **CDC Pipeline from 50 Sharded Postgres DBs into Iceberg + ClickHouse** (Debezium slots, Schema Registry, schema drift, watermarking, compaction, WAP quality checks).
   - **LLD (30 min):** Design a `WatermarkAndLatenessRouter` cleanly separating `ON_TIME`, `LATE_ALLOWED`, and `TOO_LATE_SIDE_OUTPUT` events.

---

### Days 113–120 — Advanced OLAP, Pipeline Testing, Spark Tuning & Month 4 Assessment

#### Day 113 — Query Federation vs Materialization & Pre-Aggregation (`12.4`)
1. **Pre-Requisite Learning Resource:**
   - ClickHouse Docs (*Materialized Views & Projections*) + Trino Docs (*Federated Connectors & Trade-Offs*).
2. **Hands-On Activity:**
   - **What to build:** Benchmark a live query hitting federated Postgres + Parquet via **DuckDB** (`postgres` scanner extension + `read_parquet`) vs querying a pre-materialized **ClickHouse `SummingMergeTree`** rollup (`src/olap_lab/fed_vs_mat.py`).
   - **Functions & Components:**
     - `run_federated_join()`: Joins live Postgres dimension table with 5M-row Parquet fact table; measure network/read cost.
     - `run_materialized_rollup()`: Queries ClickHouse pre-aggregated hourly MV (`SELECT ... FROM hourly_mv FINAL` or `sumMerge(...)`); record latency speedup (seconds -> single-digit milliseconds!).
   - **Verify:** Document exact criteria for when federation is acceptable (ad-hoc BI) vs when materialization is mandatory (customer-facing dashboards, >50 RPS).
3. **Daily Split:**
   - **DSA (1 hr):** LC 307 (Fenwick Tree / BIT implementation from scratch) & LC 315 (re-solve with BIT!).
   - **HLD (30 min):** Design a **Customer-Facing SaaS Analytics Dashboard** serving 5,000 QPS with `<100ms P99` latency without overloading OLTP Postgres.
   - **LLD (30 min):** Design a `MaterializedRollupManager` tracking watermark freshness of hourly/daily rollups and routing queries to raw vs pre-aggregated tables based on date range.

#### Day 114 — Apache Pinot & Star-Tree Index Concepts + Hybrid Tables (`12.4`)
1. **Pre-Requisite Learning Resource:**
   - Apache Pinot Docs (*Star-Tree Index* + *Upsert and Dedup in Real-Time Tables* + *Hybrid Offline+Realtime Tables*) + Uber Engineering Blog (*Real-time Analytics with Apache Pinot at Uber*).
2. **Hands-On Activity:**
   - **What to build:** Implement a mini Python **Star-Tree / Pre-Aggregated Cube Index** (`src/olap_lab/star_tree_mini.py`) to understand *why* Pinot answers multi-dimensional Uber Eats/ops queries in `<20ms`!
   - **Functions & Components:**
     - `StarTreeBuilder(dimensions=["city", "vehicle_type", "status"], max_leaf_records=100)`: Recursively splits nodes on dimensions when record count exceeds `max_leaf_records`, storing pre-computed `SUM`/`COUNT` aggregate nodes (`star` `*` wildcard children for skipped dimensions!).
     - `query_cube(filters)`: Traverses tree in microseconds instead of scanning millions of rows!
   - **Verify:** Compare scan record count of raw scan vs Star-Tree traversal on 100k synthetic trip records.
3. **Daily Split:**
   - **DSA (1 hr):** LC 208 (Trie) & LC 745 *Prefix and Suffix Search*.
   - **HLD (30 min):** How does Pinot's **Hybrid Table** combine an **Offline Segment Store** (S3/Iceberg batch backfill) and a **Real-Time Kafka Ingestion Segment** transparently under one SQL query?
   - **LLD (30 min):** Design a `HybridTimeBoundaryRouter` querying Offline segments for `ts < boundary_ts` and Real-time segments for `ts >= boundary_ts` with zero double-counting.

#### Day 115 — Data Pipeline Deterministic Testing & Reconciliation (`16.1`)
1. **Pre-Requisite Learning Resource:**
   - Maxime Beauchemin — *Functional Data Engineering* (`maximebeauchemin.medium.com/functional-data-engineering-a-modern-paradigm-for-batch-data-processing-2327ec32c42a`: pure/idempotent partitions, no side effects, deterministic backfills).
2. **Hands-On Activity:**
   - **What to build:** Refactor a Spark/DuckDB ETL job into a **Pure Functional, Idempotent Daily Partition Transform** (`src/etl_lab/functional_etl.py`) + automated test suite (`tests/test_etl_idempotency.py`).
   - **Functions & Components:**
     - `transform_partition(input_df, logical_date: date) -> DataFrame`: Strictly deterministic (no `now()`, no `random()`, explicit UTC window `[logical_date, logical_date + 1d)`).
     - `publish_partition_atomically(df, target_table, logical_date)`: Uses Iceberg `REPLACE PARTITION` / `DELETE + INSERT` inside one snapshot commit.
     - `test_idempotent_rerun()`: Runs `2026-11-01` twice -> asserts exact row-by-row equality and zero duplicate rows!
   - **Verify:** Also add a schema-regression test verifying column names/types match the Data Contract.
3. **Daily Split:**
   - **DSA (1 hr):** LC 399 (re-solve) & LC 990 *Satisfiability of Equality Equations* (Union-Find).
   - **HLD (30 min):** Why do non-idempotent `APPEND` pipelines corrupt data lakes on Airflow retries and backfills, and how does Functional Data Engineering + Iceberg dynamic partition overwrite solve it?
   - **LLD (30 min):** Design a `DeterministicBackfillRunner` parallelizing historical date partitions across a DAG while enforcing upstream-date dependencies.

#### Day 116 — Spark Bucketing, Dynamic Allocation & Speculative Execution (`12.2`)
1. **Pre-Requisite Learning Resource:**
   - Apache Spark Docs (*Bucketing*, *Dynamic Resource Allocation*, *Speculative Execution: `spark.speculation`*).
2. **Hands-On Activity:**
   - **What to build:** Benchmark a **Bucketed Join** (`bucketBy(16, "user_id").sortBy("user_id")`) in Spark (`src/spark_lab/bucket_join.py`) eliminating shuffles on both huge tables!
   - **Functions & Components:**
     - Save `orders_bucketed` and `users_bucketed` with identical bucket count (`16`) and join key (`user_id`).
     - Inspect `df1.join(df2, "user_id").explain("extended")` -> verify **zero `Exchange` nodes** in the physical plan!
     - Analyze when `spark.speculation=true` helps (1 slow degraded hardware node) vs hurts (false-positive duplicate writes on non-idempotent sinks or true key skew!).
   - **Verify:** Compare Shuffle Read/Write = `0 B` on bucketed join vs multi-hundred MB on unbucketed `SortMergeJoin`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 410 *Split Array Largest Sum* & LC 1231 / LC 1011 (Binary Search on Answer).
   - **HLD (30 min):** When does Spark **Dynamic Allocation** need an **External Shuffle Service / Kubernetes Push-Based Shuffle** (otherwise killing an idle executor deletes shuffle blocks needed by downstream stages!)?
   - **LLD (30 min):** Design a `SpeculativeTaskMonitor` computing median stage task duration and launching a speculative duplicate only when a task exceeds `1.5x` median AND >90% tasks finished (skipping data-skew tasks!).

#### Day 117 — ORC Format Comparison & Catalog Federation (`12.3`, `14.3`)
1. **Pre-Requisite Learning Resource:**
   - Apache ORC Spec Overview (`orc.apache.org/specification/ORCv1/` — Stripes, Row Indexes every 10k rows, Bloom filters) + Iceberg Docs (*REST Catalog Spec* & *Polaris / Unity Catalog* overview).
2. **Hands-On Activity:**
   - **What to build:** Build a tiny **Iceberg REST Catalog Metadata Explorer** (`src/lakehouse_lab/catalog_explore.py`) querying your MinIO/REST catalog HTTP endpoints (`/v1/namespaces`, `/v1/namespaces/{ns}/tables/{table}`) and comparing Parquet vs ORC (`pyarrow.orc`) compression & stripe stats.
   - **Functions & Components:**
     - `compare_parquet_vs_orc(df)`: Compares file sizes, read speeds on filtered vs full scans, and stripe/row-group stats.
     - `inspect_rest_catalog_commits()`: Calls REST Catalog endpoints via `httpx` to observe how multi-engine governance (Spark + DuckDB + Trino sharing one catalog!) locks/commits snapshots centrally.
3. **Daily Split:**
   - **DSA (1 hr):** LC 124 (re-solve) & LC 543 *Diameter of Binary Tree*.
   - **HLD (30 min):** Why is the industry moving from Hive Metastore (Thrift + RDBMS bottlenecks) to **Iceberg REST Catalog / Polaris / Unity Catalog** (vendor-neutral, multi-engine credential vending, centralized RBAC/lineage)?
   - **LLD (30 min):** Design a `CredentialVendingCatalogService` issuing short-lived scoped Azure SAS / STS tokens per table prefix when an authenticated Trino/Spark engine requests table metadata.

#### Day 118 — Streaming Joins & Interval Windows Deep Dive (`11.1`, `11.2`)
1. **Pre-Requisite Learning Resource:**
   - Apache Flink Docs (*Joins: Window Join & Interval Join*) + *DDIA* Ch. 11 (*Stream-Stream Join* example: Ad Impressions + Clicks).
2. **Hands-On Activity:**
   - **What to build:** Implement a **Stream-Stream Interval Join** in Python/PyFlink (`src/stream_sim/interval_join.py`): match `ad_impressions` with `ad_clicks` occurring within `[impression_ts, impression_ts + 30 minutes]` while expiring unmatched impressions via RocksDB/state TTL!
   - **Functions & Components:**
     - `IntervalJoinBuffer`: Maintains two indexed state maps (`left_buffer[key]`, `right_buffer[key]`).
     - `on_left(impression)`: Joins against existing `right` entries in `[ts, ts + 30m]`, stores impression until `watermark > ts + 30m`.
     - `on_right(click)`: Joins against existing `left` entries in `[ts - 30m, ts]`, stores click until `watermark > ts`.
     - `on_watermark(wm)`: Evicts expired entries cleanly so memory stays bounded!
   - **Verify:** Test clicks arriving *before* impressions (out-of-order network!), late clicks inside 30m, and clicks at 35m (no false match!).
3. **Daily Split:**
   - **DSA (1 hr):** LC 986 & LC 1288 *Remove Covered Intervals*.
   - **HLD (30 min):** Design **Real-Time Ad Click Attribution** (1M impressions/sec + 50k clicks/sec, 30-min attribution window, deduplication of double-clicks, RocksDB memory sizing math!).
   - **LLD (30 min):** Add **Left-Outer Join** emission ("emit unclicked impression after 30m timer expires") to `IntervalJoinBuffer`.

#### Day 119 — Month 4 Cross-Domain Integration: End-to-End Lakehouse & OLAP Mini-Platform
1. **Pre-Requisite Learning Resource:**
   - Review your Month 4 `notes.md` across Debezium, Flink, Spark, Parquet, Iceberg, ClickHouse, and DuckDB.
2. **Hands-On Activity:**
   - **What to build:** Run a clean architectural simulation script / benchmark on Ubuntu VM comparing **all 4 storage/compute patterns** on one problem ("Hourly revenue + unique active users per tenant"):
     1. OLTP Postgres (`EXPLAIN ANALYZE` on 5M rows).
     2. Spark Batch -> Iceberg Parquet (`Spark UI` stage metrics).
     3. DuckDB direct query on that Iceberg Parquet (`EXPLAIN ANALYZE`).
     4. ClickHouse `MergeTree` + `AggregatingMergeTree` (`uniqState` HyperLogLog!).
   - **Verify:** Record a clean comparison table in `docs/engine_benchmarks.md` with exact latencies, RAM used, and when to choose each engine.
3. **Daily Split:**
   - **DSA (1 hr):** Timed Mock: LC 42 *Trapping Rain Water* + LC 207 *Course Schedule*.
   - **HLD (30 min):** Timed 30-min Blank-Page Design: **Multi-Tenant Product Analytics Platform (Amplitude/Mixpanel-lite)** (ingestion Kafka -> ClickHouse + Iceberg, funnel & retention queries, noisy-tenant isolation).
   - **LLD (30 min):** Design a `HyperLogLog` wrapper or `TenantQueryRouter` enforcing per-tenant concurrency & memory quotas across ClickHouse/Trino.

#### Day 120 — Month 4 Comprehensive Assessment
1. **Pre-Requisite Learning Resource:**
   - Full audit of [Curriculum.md](./Curriculum.md) Sections `10.4`, `11.1–11.3`, `12.1–12.4`, `14.3`.
2. **Hands-On Activity:**
   - **What to build:** Clean-room 90-min test (zero AI):
     1. Diagnose a skewed Spark join + fix it with salting & AQE.
     2. Explain out loud without notes: Debezium replication slot risk -> Flink Chandy-Lamport barriers + RocksDB -> Parquet Row Group pruning -> Iceberg snapshot isolation -> ClickHouse sparse primary index.
     3. Confirm Ubuntu VM containers are pruned (`docker compose down`) leaving >14 GB free RAM for Month 5.
3. **Daily Split:**
   - **DSA (1 hr):** Timed Mock (unseen): LC 84 *Largest Rectangle in Histogram* + LC 1631 *Path With Minimum Effort*.
   - **HLD (30 min):** Timed 30-min Blank-Page Design: **Uber Driver Earnings & Trip Batch/Stream Reconciliation Platform** (Flink real-time balance + Spark/Iceberg nightly audit + WAP quality gates).
   - **LLD (30 min):** Timed 30-min Blank-Page Design: **Event-Time Sliding Window Aggregator with Bounded Out-of-Order Watermarks and Side-Output**.
