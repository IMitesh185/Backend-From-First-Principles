# Backend & Data Platform Engineering — 5-Month Deep Execution Plan (150 Days)

> **Companion to:** [Curriculum.md](./Curriculum.md) & [About.md](./About.md)
> **Hardware & Cloud Lab:** Main Workstation + Separate Ubuntu VM (16 GB RAM, 512 GB SSD) + Azure Free Tier
> **Daily Structure (~4–5 hrs/day):** Pre-Requisite Study + Hands-On Engineering Build (2–3 hrs) + **1 hr DSA + 30 min HLD + 30 min LLD**

---

## Monthly Volumes (All 150 Days Individually Specified)

- **[Month 1 (Days 1–30)](./5_Month_Plan/Month_1_Days_001_to_030.md):** Python OOP Beyond `for`-Loops, `pytest`/Interfaces, Linux/Sockets/HTTP, `localhost` -> Live Public HTTPS on Azure, Nginx/TLS 1.3, Docker & CI/CD
- **[Month 2 (Days 31–60)](./5_Month_Plan/Month_2_Days_031_to_060.md):** OS Memory/`epoll`, CPython Internals/GIL/`asyncio`/Profiling, Mini LSM Storage Engine, PostgreSQL MVCC/`EXPLAIN`/Locks/`PgBouncer`, Redis, Job Queues, Webhooks & Elasticsearch
- **[Month 3 (Days 61–90)](./5_Month_Plan/Month_3_Days_061_to_090.md):** Distributed Systems Theory (Clocks, Consistency, Quorums, Raft, Fencing Tokens, Outbox/Sagas, NoSQL/NewSQL), Apache Kafka Internals (KRaft, ISR, Idempotence, Rebalances) & Object Storage
- **[Month 4 (Days 91–120)](./5_Month_Plan/Month_4_Days_091_to_120.md):** Debezium CDC, Apache Flink Stream Processing (Watermarks, RocksDB, Chandy-Lamport), Apache Spark Internals (Catalyst, Tungsten, AQE, Shuffle, Skew), Parquet/Iceberg/Delta/Hudi, DuckDB/ClickHouse/Trino & Data Governance
- **[Month 5 (Days 121–150)](./5_Month_Plan/Month_5_Days_121_to_150.md):** Auth/OAuth2/JWT/OWASP/KMS, Kubernetes (`k3s`), Terraform, Temporal, OpenTelemetry/Prometheus, `k6` `P99` Tuning, Resilience/Chaos, Tier-1 Primitives (Sketches, Ring Buffers, Spatial Indexing), Big Tech & 50 LPA+ Startup Architectures (YouTube, Amazon, LinkedIn, Buffer, Ledger, Multi-Region) & Mock Loops

---

## Daily 3-Part Format Rules

Every single day from Day 1 to Day 150 contains:
1. **Pre-Requisite Learning Resource:** Exact book chapters/subsections, RFCs, official docs, source files, papers, or high-signal engineering blogs—never 2-hour YouTube playlists, never W3Schools/tutorialspoint fluff, never vague "read Book X".
2. **Hands-On Activity (No Code — Architecture, Functions & What Each Contains):** Exact modules, classes, functions, responsibilities, edge cases, and terminal verification/breakage checks to build yourself on your Ubuntu 16 GB VM or Azure VM.
3. **Daily 2-Hour Interview Split (`1 hr DSA` | `30 min HLD` | `30 min LLD`):**
   - **DSA (60 min):** Specific problem(s), target pattern, and time/space constraint—coded from scratch without AI/autocomplete.
   - **HLD (30 min):** Architectural diagram + capacity/bottleneck math + failure modes on a blank page.
   - **LLD (30 min):** Class, interface (`Protocol`), state-machine, or concurrency design spec (signatures, invariants, thread-safety).

---

## Lab Discipline (Ubuntu 16 GB VM + Azure Free Tier)

### Ubuntu VM (16 GB RAM, 512 GB SSD)
- Always SSH from your main laptop (`~/.ssh/config`). Never treat it as `localhost`.
- **16 GB RAM Rule:** Run **one heavy stack at a time** (e.g., do not run Kafka + Flink + Spark + ClickHouse simultaneously). Cap JVMs (`-Xms512m -Xmx1g` or `2g`), set Docker `mem_limit`, and run `docker compose down` between domain shifts.

### Azure Free Tier
- **B1s Linux VM** (1 vCPU, 1 GB RAM, 750 hrs/mo free for 12 months): public edge for Month 1 (`systemd`, DNS, Nginx, Let's Encrypt TLS).
- **Azure Blob Storage** (5 GB free) + **Key Vault** + **Managed Identity** + **ACR**: used in Months 3–5.
- **Cost Safety:** Set a **$5 billing budget alert** on Day 1. Never leave NAT Gateway, Azure Firewall, Application Gateway, or multi-node AKS clusters running.

### No-AI-Crutch Rule
- Never ask AI to generate the day's functions, configs, or DSA solutions. Write your draft first, run it, inspect logs/metrics, and use AI strictly to critique your draft or diagnose errors you already reproduced.
