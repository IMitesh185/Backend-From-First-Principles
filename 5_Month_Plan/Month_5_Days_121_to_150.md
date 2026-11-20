# Month 5 (Days 121–150): Security, Kubernetes, Temporal, Observability, SRE, Tier-1 Architectures & Mock Loops

> **Curriculum Coverage:** `13.1–13.3`, `14.1–14.2`, `15.2`, `15.3` (Terraform/Cloud), `15.4`, `16.1–16.2`, `17.1–17.4`
> **Environment:** Main Laptop + Ubuntu 16 GB VM (`k3s`, Prometheus, Grafana, Jaeger/Tempo, Temporal, `k6`) + Azure Free Tier (Key Vault, Managed Identity, Terraform)

---

### Day 121 — Password Hashing, JWT Internals & Refresh Token Rotation (`14.1`)
1. **Pre-Requisite Learning Resource:**
   - RFC 7519 (*JSON Web Token (JWT)* §1–4) + Auth0 / PortSwigger Blog (*Critical Vulnerabilities in JSON Web Token Libraries*: `alg=none`, `RS256->HS256` key confusion) + OWASP Password Storage Cheat Sheet (`Argon2id`).
2. **Hands-On Activity:**
   - **What to build:** Build a production-grade Auth Service module (`src/auth/tokens.py`) using `argon2-cffi` + asymmetric `RS256` JWTs + rotating opaque refresh tokens in Postgres.
   - **Functions & Components:**
     - `hash_password(plain)` / `verify_password(plain, stored_hash)`: Uses `Argon2id` (salted automatically, constant-time verify) + optional application-level pepper from env.
     - `JwtIssuer` & `JwtVerifier`: Signs short-lived (10 min) access JWTs with `kid` header (`RS256` private key); `JwtVerifier` fetches public keys from `/auth/.well-known/jwks.json`, **explicitly hardcodes `algorithms=["RS256"]`** (never trusting header `alg`!), and validates `iss`, `aud`, `exp`, `nbf`.
     - `rotate_refresh_token(old_token)`: Stores SHA-256 hash of opaque refresh token in Postgres; on refresh, revokes `old_token` and issues a new token in the same family—if an already-revoked token is replayed (**theft detection!**), revokes the entire token family!
   - **Break Intentionally:** Craft a forged JWT with `{"alg": "none"}` and a token signed with `HS256` using the public PEM string -> prove `JwtVerifier` rejects both!
3. **Daily Split:**
   - **DSA (1 hr):** LC 208 (Trie) & LC 648 *Replace Words* & LC 676 *Implement Magic Dictionary*.
   - **HLD (30 min):** Contrast **Opaque Reference Tokens** (instant revocation via Redis lookup, network hop per check) vs **Short-Lived JWTs + Refresh Rotation** (zero DB lookup on verify, bounded 10-min revocation window).
   - **LLD (30 min):** Design a `JWKSKeyRotator` supporting overlapping active/previous RSA keys (`kid_1`, `kid_2`) during zero-downtime key rotation.

---

### Day 122 — OAuth 2.0 (PKCE, Client Credentials) & OpenID Connect (`14.1`)
1. **Pre-Requisite Learning Resource:**
   - RFC 7636 (*Proof Key for Code Exchange (PKCE)* §1, §4) + OAuth 2.0 Simplified (`oauth.net/2/`) + OpenID Connect Core 1.0 (*ID Token* §2).
2. **Hands-On Activity:**
   - **What to build:** Implement a minimal **OAuth 2.0 Authorization Code + PKCE** flow (`src/auth/oauth_pkce.py`) in Python to demystify every redirect, `state`, `code_verifier`, and `code_challenge`.
   - **Functions & Components:**
     - `ClientApp`: Generates random `code_verifier`, computes `code_challenge = base64url(sha256(code_verifier))`, generates CSRF `state`, redirects to `/authorize`.
     - `AuthServer`: Issues single-use 60s `authorization_code` bound to `(client_id, redirect_uri, code_challenge)`; at `POST /token`, verifies `base64url(sha256(code_verifier)) == stored_challenge` before issuing Access Token + OIDC ID Token.
     - `client_credentials_exchange(client_id, client_secret)`: Service-to-service M2M token issuance.
   - **Break Intentionally:** Intercept `authorization_code` and call `/token` with a wrong `code_verifier` -> confirm `400 invalid_grant`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 131 & LC 39 & LC 40 *Combination Sum II*.
   - **HLD (30 min):** Trace the full OAuth2 + OIDC login sequence across Browser -> Client App -> Identity Provider (IdP) -> Resource API, explaining why Authorization Code + PKCE replaced the deprecated Implicit flow.
   - **LLD (30 min):** Design an `AuthorizationCodeStore` enforcing single-use consumption (revoking tokens if an auth code is replayed twice!) and strict exact-match `redirect_uri` validation.

---

### Day 123 — Authorization Models: RBAC, ABAC & Zanzibar ReBAC (`14.1`)
1. **Pre-Requisite Learning Resource:**
   - Google Research Paper — *Zanzibar: Google's Consistent, Global Authorization System* (§1–3: Relation Tuples, Usersets, Zookies) + PostgreSQL Docs (*Row Security Policies (RLS)* §5.8).
2. **Hands-On Activity:**
   - **What to build:** Implement a mini **Zanzibar-Style ReBAC Tuple Evaluator** in Python (`src/auth/zanzibar_mini.py`) AND test **Postgres Row-Level Security (RLS)** for multi-tenant isolation.
   - **Functions & Components:**
     - `RelationTuple(object_type, object_id, relation, subject)` (e.g., `folder:eng#viewer@group:backend#member` and `doc:spec1#parent@folder:eng`).
     - `check_permission(subject, relation, obj, visited: set) -> bool`: Recursively evaluates direct tuples, computed relations (`owner -> editor -> viewer`), and userset rewrites (`parent->viewer`) with cycle detection!
     - **Postgres RLS:** Enable `ALTER TABLE invoices ENABLE ROW LEVEL SECURITY; CREATE POLICY tenant_isolation ON invoices USING (tenant_id = current_setting('app.current_tenant')::uuid);` -> prove a buggy query missing `WHERE tenant_id = ...` still returns **0 cross-tenant rows**!
   - **Verify:** Test nested group membership (`alice -> group:core -> folder:eng -> doc:spec1`) and RLS isolation across two tenant sessions.
3. **Daily Split:**
   - **DSA (1 hr):** LC 785 & LC 886 *Possible Bipartition* & Graph reachability cycle checks.
   - **HLD (30 min):** When is **RBAC** enough, when do you need **ABAC** (time/IP/clearance rules), and why do Google Drive / Carta / Slack-scale apps adopt **Zanzibar ReBAC**? Also explain Zanzibar's **Zookie** consistency token (fixing the New Enemy problem!).
   - **LLD (30 min):** Design a `PermissionCheckCache` invalidating cached `(subject, relation, obj)` decisions safely when tuple edges change.

---

### Day 124 — OWASP API Top 10 Hands-On Audit (`14.2`)
1. **Pre-Requisite Learning Resource:**
   - OWASP API Security Top 10 (2023) (`owasp.org/API-Security/editions/2023/en/0x11-t10/` — focus on API1 BOLA, API2 Auth, API3 Property Level Auth/Mass Assignment, API4 Unrestricted Resource Consumption, API5 BFLA, API7 SSRF).
2. **Hands-On Activity:**
   - **What to build:** Write an automated **Security Regression Test Suite** (`tests/security/test_owasp.py`) attacking your own FastAPI Order/URL Shortener service and fixing every hole found!
   - **Functions & Components:**
     - `test_bola_idor()`: Authenticate as User A, attempt `GET /v1/orders/{user_b_order_id}` -> assert `403`/`404` (never `200`!).
     - `test_bfla_admin_route()`: Authenticate as standard user, attempt `POST /v1/admin/users/{id}/disable` -> assert `403 Forbidden`.
     - `test_mass_assignment()`: Send `{"long_url": "...", "is_admin": true, "role": "superuser"}` -> verify strict Pydantic DTO ignores/rejects privileged fields.
     - `test_ssrf_webhook_guard()`: Attempt registering webhook URL `http://169.254.169.254/latest/meta-data/` or `http://127.0.0.1:8000/admin` -> implement DNS-resolution + IP-range blocker (`ipaddress.ip_address(resolved).is_private | is_loopback | is_link_local`) rejecting private targets!
   - **Verify:** Watch out for **DNS Rebinding** on SSRF checks (resolve IP once, validate it is public, and connect directly to that validated IP!).
3. **Daily Split:**
   - **DSA (1 hr):** LC 93 *Restore IP Addresses* & LC 468 *Validate IP Address*.
   - **HLD (30 min):** How do you safely fetch user-supplied webhook URLs or image URLs in production without exposing internal VPC services or cloud metadata (`169.254.169.254`)? (Egress proxy in isolated DMZ + IP validation + disabling HTTP redirects to internal IPs).
   - **LLD (30 min):** Design an `SSRFSafeHttpFetcher` resolving hostname -> rejecting private/reserved/multicast CIDRs -> connecting directly with `Host` header and blocking unsafe redirects.

---

### Day 125 — KMS Envelope Encryption, Cloud Secrets & Supply-Chain Security (`14.2`, `15.3`)
1. **Pre-Requisite Learning Resource:**
   - AWS KMS / Azure Key Vault Docs (*Envelope Encryption: DEK vs KEK*) + HashiCorp Vault Docs (*Dynamic Secrets Overview*) + SLSA / Docker Docs (*SBOMs & Image Signing*).
2. **Hands-On Activity:**
   - **What to build:** Implement **Envelope Encryption** (`src/security/envelope.py`) using AES-256-GCM + **Azure Key Vault & Managed Identity** on your Azure B1s VM!
   - **Functions & Components:**
     - `EnvelopeEncryptor`: For a sensitive payload: generates a fresh 256-bit random **Data Encryption Key (DEK)** (`os.urandom(32)`), encrypts plaintext with `AESGCM(dek)` + unique 96-bit nonce, wraps/encrypts the DEK using the remote **Key Encryption Key (KEK)** in Azure Key Vault (`CryptographyClient.wrap_key`), and stores `{wrapped_dek, nonce, ciphertext, kek_version}`.
     - `decrypt_record()`: Calls Key Vault `unwrap_key(wrapped_dek)` -> decrypts locally in RAM with AES-GCM.
     - **Azure Managed Identity:** Enable System-Assigned Managed Identity on your Azure B1s VM, grant it `Key Vault Crypto User` + `Secrets User` RBAC, and fetch a secret with `DefaultAzureCredential` (**zero hardcoded API keys on disk!**).
   - **Verify:** Generate an SBOM (`syft` or `pip-audit`) for your Docker image to inspect supply-chain vulnerabilities (`14.2`).
3. **Daily Split:**
   - **DSA (1 hr):** LC 49 & LC 128 & LC 560 (hash/prefix speed drills).
   - **HLD (30 min):** Why do cloud systems use **Envelope Encryption (DEK + KEK)** instead of sending gigabytes of database rows directly to KMS over the network? Also explain how rotating a KEK avoids re-encrypting terabytes of data.
   - **LLD (30 min):** Design a `LocalDEKCache` caching unwrapped DEKs in memory with short TTL and key-version rotation support to reduce KMS API calls and rate-limit costs.

---

### Day 126 — Kubernetes Architecture (`k3s`): Control Plane, Nodes & Pod Probes (`15.2`)
1. **Pre-Requisite Learning Resource:**
   - Kubernetes Official Docs (*Cluster Architecture: Control Plane & Node Components*, *Pod Lifecycle*, *Configure Liveness, Readiness and Startup Probes*).
2. **Hands-On Activity:**
   - **What to build:** Install lightweight **`k3s`** on your Ubuntu 16 GB VM (single-node cluster using ~500 MB RAM) and deploy your FastAPI app (`k8s/api-deployment.yaml`).
   - **Functions & Components:**
     - Inspect system components (`kubectl get pods -n kube-system`, `kubectl get nodes -o wide`).
     - Write YAML manifests from scratch (no Helm yet!): `Namespace`, `ConfigMap`, `Secret`, `Deployment` (2 replicas), `ResourceRequests/Limits` (`requests: 128Mi/100m`, `limits: 256Mi/500m`), `startupProbe` (`/health/live`), `readinessProbe` (`/health/ready`), `livenessProbe` (`/health/live`), and `lifecycle.preStop` (`sleep 5` so kube-proxy/Endpoints remove the pod *before* `SIGTERM` arrives!).
   - **Break Intentionally:**
     1. Set `livenessProbe` to hit a nonexistent `/wrong` path -> watch `CrashLoopBackOff` in `kubectl describe pod`.
     2. Lower memory limit to `32Mi` -> watch `OOMKilled` (`Exit Code 137`) and `kubectl logs --previous`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 200 & LC 130 & LC 994 (Graph BFS/DFS speed drills).
   - **HLD (30 min):** Walk through what happens inside Kubernetes when you run `kubectl apply -f deployment.yaml`: `kube-apiserver` -> `etcd` -> `DeploymentController` -> `ReplicaSetController` -> `kube-scheduler` -> `kubelet` -> `containerd` -> `runc`.
   - **LLD (30 min):** Design a Kubernetes-style `ReconciliationLoop` (`desired_replicas`, `observe_actual()`, `diff_and_act()`) that is idempotent and level-triggered (not edge-triggered!).

---

### Day 127 — Kubernetes Workloads: `StatefulSet`, `DaemonSet`, `Job` & `PDB` (`15.2`)
1. **Pre-Requisite Learning Resource:**
   - Kubernetes Docs (*StatefulSets*, *DaemonSet*, *Jobs*, *Disruptions & PodDisruptionBudgets*).
2. **Hands-On Activity:**
   - **What to build:** Deploy a 2-replica **`StatefulSet`** (e.g., Redis or Postgres/custom stateful worker with `volumeClaimTemplates`) + a **`PodDisruptionBudget` (`minAvailable: 1`)** + a **`CronJob`** on `k3s` (`k8s/stateful_and_pdb.yaml`).
   - **Functions & Components:**
     - `Headless Service` (`clusterIP: None`) giving stable DNS `worker-0.worker-headless`, `worker-1.worker-headless`.
     - `StatefulSet` with `volumeClaimTemplates` (`local-path` storage in `k3s`): delete `worker-0` and observe K8s recreates `worker-0` attached to the *exact same* PersistentVolumeClaim!
     - `PodDisruptionBudget`: `minAvailable: 1`; test `kubectl drain` simulation/eviction behavior.
   - **Verify:** Compare Pod naming, startup ordering, and storage binding between `Deployment` vs `StatefulSet`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 127 & LC 433 *Minimum Genetic Mutation* (BFS shortest state transition).
   - **HLD (30 min):** Why must Kafka/ZooKeeper/Postgres run on `StatefulSet` + Headless Service + `PDB` instead of `Deployment`, and what happens during an ungraceful node failure (`StatefulSet` stuck `Terminating` pitfall)?
   - **LLD (30 min):** Design a `ClusterDrainer` respecting `PodDisruptionBudgets` before cordoning and evicting pods from a node during rolling upgrades.

---

### Day 128 — Kubernetes Networking, Ingress/Gateway API & Scheduling (`15.2`)
1. **Pre-Requisite Learning Resource:**
   - Kubernetes Docs (*Service, Load Balancing, and Networking*: `ClusterIP`, `NodePort`, `Ingress`, `Gateway API`, `IPVS` vs `iptables`) + *Assigning Pods to Nodes* (Taints/Tolerations, Affinity, `topologySpreadConstraints`) + Cilium eBPF overview (`[T3]`).
2. **Hands-On Activity:**
   - **What to build:** Wire `ClusterIP` + `Ingress` (Traefik built into `k3s`) routing `/v1/urls` to your FastAPI Service, inspect `iptables` / `IPVS` NAT rules on the host, and configure **Topology Spread Constraints & Taints** (`k8s/net_and_schedule.yaml`).
   - **Functions & Components:**
     - `Ingress` routing `api.local` -> `api-svc:80` -> Pod port `8000`.
     - Add `podAntiAffinity` + `topologySpreadConstraints` (`topologyKey: kubernetes.io/hostname`, `maxSkew: 1`) so replicas never land on the same node/AZ when multi-node!
     - Taint your node (`kubectl taint nodes ... dedicated=spark:NoSchedule`), observe pending Pods (`kubectl describe pod` showing `0/1 nodes available: untolerated taint`), add matching `toleration`, and untaint afterward.
   - **Verify:** Test CoreDNS inside a debug pod (`kubectl run tmp --rm -it --image=busybox -- nslookup api-svc`).
3. **Daily Split:**
   - **DSA (1 hr):** LC 1584 & LC 743 & LC 787 (Weighted graph + MST/Shortest Path).
   - **HLD (30 min):** Why does `kube-proxy` `iptables` mode degrade at 10,000+ Kubernetes Services ($O(N)$ sequential rule traversal on every packet) and how do `IPVS` ($O(1)$ hash table) and **Cilium eBPF** (bypassing `iptables` directly in kernel socket/XDP hooks) solve it?
   - **LLD (30 min):** Design a `KubernetesSchedulerFilterAndScore` (`filter_nodes(pod, nodes)`, `score_nodes(pod, feasible_nodes)`) honoring CPU/RAM fit, taints, and spread balance.

---

### Day 129 — Kubernetes Autoscaling (`HPA`/`VPA`/`KEDA`) & Operators (`15.2`)
1. **Pre-Requisite Learning Resource:**
   - Kubernetes Docs (*Horizontal Pod Autoscaling* + *Custom Resources & Operator Pattern*) + KEDA Docs (*Event-Driven Autoscaling: Kafka Lag Scaler* `[T2]`).
2. **Hands-On Activity:**
   - **What to build:** Configure an **`HPA`** on your FastAPI `Deployment` (`minReplicas: 1`, `maxReplicas: 5`, target CPU `50%`) and trigger real scale-out & scale-down stabilization on `k3s` (`k8s/hpa.yaml`).
   - **Functions & Components:**
     - Deploy `HPA` targeting CPU utilization via `metrics-server` (built into `k3s`).
     - Generate sustained CPU load via `k6` -> watch `kubectl get hpa -w` scale from `1 -> 3 -> 5` pods, then stop load and observe the 5-minute default scale-down stabilization window preventing thrashing!
     - Study **Operators (`[T3]`)**: How `Strimzi` (Kafka), `Spark Operator`, and `Flink Operator` extend K8s with CRDs + custom reconciliation controllers.
   - **Verify:** Contrast `HPA` (replicas out/in on CPU/RPS), `VPA` (resizing pod requests—usually restarts pods!), and `KEDA` (scaling workers from 0->N on Kafka consumer lag or Postgres queue depth!).
3. **Daily Split:**
   - **DSA (1 hr):** LC 684 & LC 721 (Union-Find drills).
   - **HLD (30 min):** Why is scaling Kafka consumers purely on **CPU (`HPA`)** broken when consumers are I/O-bound waiting on slow downstream DB calls, and how does **KEDA scaling on Consumer Lag** solve it (respecting `max_replicas == partition_count`)?
   - **LLD (30 min):** Design an `AutoscalerStabilizer` implementing desired-replica computation (`ceil(current * metric / target)`) with scale-up tolerance band (`±10%`) and scale-down cooldown window.

---

### Day 130 — Cloud Networking & Terraform IaC on Azure (`15.3`)
1. **Pre-Requisite Learning Resource:**
   - HashiCorp Terraform Docs (*State*, *State Locking*, *Modules*, *Drift Detection*) + Azure Architecture Center (*Hub-and-Spoke VNet Topology & Egress Cost Considerations*).
2. **Hands-On Activity:**
   - **What to build:** Write a clean modular **Terraform** configuration (`infra/azure/`) provisioning an isolated Azure Resource Group + VNet/Subnet + Storage Account + Key Vault—inspect `plan`, `apply`, drift detection, and **`destroy`** cleanly!
   - **Functions & Components:**
     - `modules/storage_vault/`: Parameterized `variables.tf`, `main.tf`, `outputs.tf` (no hardcoded environment names!).
     - `envs/dev/main.tf`: Calls module, creates resources cleanly (`terraform apply`).
     - **Drift Drill:** Manually add a tag or change a Storage tier in Azure Portal -> run `terraform plan` and observe Terraform detecting external configuration drift!
     - **Cleanup:** Run `terraform destroy` immediately so zero paid cloud resources linger.
   - **Verify:** Inspect `terraform.tfstate` JSON to understand why remote state locking (e.g., Azure Blob lease lock) is mandatory for teams.
3. **Daily Split:**
   - **DSA (1 hr):** LC 146 & LC 460 (re-solve LRU + LFU back-to-back).
   - **HLD (30 min):** Sketch a production Multi-AZ Cloud VPC/VNet (Public Subnets for NAT/ALB, Private App Subnets, Isolated DB Subnets, VPC Endpoints/PrivateLink to Object Storage so traffic never pays NAT Gateway $0.045/GB processing fees!).
   - **LLD (30 min):** Design a `ResourceDependencyGraphExecutor` building a DAG from Terraform resource references, validating zero cycles, and executing creates/destroys in topological order.

---

### Day 131 — DAG Orchestrators (Airflow / Dagster) & Internal Developer Platforms (`15.4`)
1. **Pre-Requisite Learning Resource:**
   - Apache Airflow Docs (*Architecture Overview: Scheduler, Executors, DAGs, Dynamic Task Mapping, Deferrable Operators/Triggers*) + Dagster Docs (*Software-Defined Assets*).
2. **Hands-On Activity:**
   - **What to build:** Inspect DAG scheduler mechanics by building a lightweight **Python Async DAG Orchestrator** (`src/orch_lab/dag_engine.py`) with cycle detection, dynamic task mapping, sensor/trigger scaling, and backfill!
   - **Functions & Components:**
     - `DAG.validate()`: Uses Kahn's topological sort to reject cyclic dependencies at parse time.
     - `DAGScheduler.run(logical_date, max_parallel=4)`: Executes ready tasks (all upstream parents `SUCCESS`), supports dynamic fan-out (`expand(items)`), and simulates **Deferrable Triggerer** (1 async coroutine polling 1,000 sensors instead of 1,000 idle worker slots!).
   - **Verify:** Contrast Airflow's task-centric model vs Dagster's asset-lineage model + how golden-path IDP templates reduce startup cognitive load.
3. **Daily Split:**
   - **DSA (1 hr):** LC 210 & LC 1136 & LC 2050 *Parallel Courses III* (weighted DAG longest-path DP).
   - **HLD (30 min):** Why does Airflow's scheduler choke when 500 sensors occupy 500 Celery/Kubernetes worker slots polling S3/APIs, and how do **Deferrable Operators + Triggerer (`asyncio`)** fix it?
   - **LLD (30 min):** Design a `BackfillStateStore` tracking per-`(dag_id, logical_date, task_id)` states (`PENDING`, `RUNNING`, `SUCCESS`, `UPSTREAM_FAILED`) across concurrent workers.

---

### Day 132 — Durable Execution Engines: Temporal Internals (`15.4`)
1. **Pre-Requisite Learning Resource:**
   - Temporal Official Docs (*What is a Temporal Workflow?*, *Event History & Deterministic Replay Constraints*, *Activities & Task Queues*).
2. **Hands-On Activity:**
   - **What to build:** Install Temporal CLI (`temporal server start-dev`) on Ubuntu VM and build a multi-step **Order Fulfillment Saga** using the Python Temporal SDK (`src/temporal_lab/order_workflow.py`).
   - **Functions & Components:**
     - `OrderWorkflow`: Calls Activities `reserve_inventory`, `charge_card`, `ship_order`; on failure runs compensation activities; uses `workflow.now()` and `workflow.uuid4()` (NEVER Python `datetime.now()` or `random` inside workflow code!).
     - `Worker`: Polls Temporal Task Queue.
   - **Break Intentionally:**
     1. Kill the Python `Worker` (`kill -9`) **while `OrderWorkflow` is sleeping or between activities**, restart the worker, and watch **Temporal Event History Replay** seamlessly resume from the exact step without re-charging the card!
     2. Put a non-deterministic `random.random()` branch inside workflow code -> trigger Temporal's **`NonDeterminismError`** and understand why!
3. **Daily Split:**
   - **DSA (1 hr):** LC 139 & LC 140 *Word Break II* (memoized DFS).
   - **HLD (30 min):** Compare **Airflow** (batch data DAGs on schedule) vs **Temporal** (millions of long-lived entity workflows/sagas with code-as-state durable replay) vs **Postgres `SKIP LOCKED` Queue** (simple async jobs).
   - **LLD (30 min):** Design a tiny `DurableReplayEngine` storing `EventHistory` (`ActivityScheduled`, `ActivityCompleted(result)`) and re-executing a deterministic generator function, short-circuiting already-completed steps from history!

---

### Day 133 — Observability: Structured Logging & Prometheus Metrics (`17.1`)
1. **Pre-Requisite Learning Resource:**
   - Prometheus Docs (*Metric Types: Counter, Gauge, Histogram, Native Histograms*, *Recording Rules*, *Instrumentation Best Practices: Cardinality*) + Brendan Gregg (*USE Method*) & Tom Wilkie (*RED Method*).
2. **Hands-On Activity:**
   - **What to build:** Run **Prometheus + Grafana** on Ubuntu VM (Docker or inside `k3s`) scraping your FastAPI app (`src/obs_lab/metrics.py`).
   - **Functions & Components:**
     - `StructuredLogFormatter`: Outputs single-line JSON `{ts, level, msg, request_id, trace_id, span_id}` with an automatic **PII Redactor** stripping `Authorization`, `password`, and email/card patterns!
     - `PrometheusMiddleware`: Exports RED metrics (`http_requests_total{method, route_template, status_code}` Counter, `http_request_duration_seconds` Histogram with tuned SLO buckets `[0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5]`, `http_in_flight_requests` Gauge).
     - Grafana Panel queries: `histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route_template))`.
   - **Break Intentionally:** Label a metric with raw `path` (`/v1/urls/abc123`) or `user_id` instead of normalized `route_template` (`/v1/urls/{code}`) -> watch Prometheus active series explode (**High-Cardinality Pitfall!**) and fix it.
3. **Daily Split:**
   - **DSA (1 hr):** LC 295 (Two Heaps) & LC 703 (Stream Kth Largest).
   - **HLD (30 min):** Why can't you average `P99` latencies across 10 servers (`avg(p99)` is mathematically invalid!), and why do Histograms (or Native Histograms / DDSketch) allow correct cluster-wide `P99` aggregation via Prometheus `histogram_quantile`?
   - **LLD (30 min):** Design a thread-safe `HDRHistogram` / `BucketedHistogram` tracking cumulative bucket counts without storing individual request samples.

---

### Day 134 — Distributed Tracing (OpenTelemetry) & Continuous Profiling (`17.1`)
1. **Pre-Requisite Learning Resource:**
   - OpenTelemetry Docs (*Concepts: Traces, Spans, Context Propagation, OTel Collector, Tail-Based Sampling*) + W3C Trace Context Spec (`w3.org/TR/trace-context/` — `traceparent` header).
2. **Hands-On Activity:**
   - **What to build:** Wire **OpenTelemetry SDK + OTel Collector + Jaeger/Grafana Tempo** across **FastAPI -> Postgres -> Redis -> Kafka Producer -> Kafka Consumer** on Ubuntu VM!
   - **Functions & Components:**
     - Instrument FastAPI, `psycopg`, and `redis-py` with OTel auto/manual spans (`with tracer.start_as_current_span("validate_url"):`).
     - **Manual Context Propagation Over Kafka:** Inject current `traceparent` into Kafka message headers on produce, extract `traceparent` in the async consumer so one unified trace spans synchronous HTTP + asynchronous Kafka processing!
     - Configure OTel Collector pipeline (`otlp receiver -> batch processor -> tail_sampling processor (keep 100% errors & >500ms slow traces, sample 5% healthy!) -> jaeger exporter`).
   - **Verify:** Open Jaeger UI and inspect the exact waterfall trace showing HTTP + Redis miss + Postgres query + Kafka publish + async consumer lag!
3. **Daily Split:**
   - **DSA (1 hr):** LC 124 & LC 297 (Tree path/serialization speed drills).
   - **HLD (30 min):** Contrast **Head-Based Sampling** (decide 1% at API edge -> loses rare `500` errors!) vs **Tail-Based Sampling** (buffer full trace in Collector by `trace_id` routing, keep all errors & `P99` spikes). Also explain continuous eBPF profiling (`Parca`/`Pyroscope`).
   - **LLD (30 min):** Design a `TraceContextPropagator` parsing/formatting W3C `traceparent: 00-<32hex_trace_id>-<16hex_span_id>-<flags>` and propagating it across `asyncio` tasks via `contextvars`.

---

### Day 135 — Load Testing (`k6`), Coordinated Omission & Traffic Shadowing (`16.1`, `16.2`)
1. **Pre-Requisite Learning Resource:**
   - Gil Tene — *How NOT to Measure Latency* (Coordinated Omission!) + Grafana `k6` Docs (*Arrival-Rate Executors: Open vs Closed Load Models* + *Load/Stress/Soak/Spike Tests*).
2. **Hands-On Activity:**
   - **What to build:** Build a **`k6` Open-Loop vs Closed-Loop Benchmark Suite** (`tests/load/api_load.js`) against your `k3s`/Docker API + a lightweight **Traffic Shadowing (Dark Launch)** proxy test (`src/obs_lab/shadow_diff.py`).
   - **Functions & Components:**
     - `k6` Closed-Loop (`vus: 50`): Sleeps waiting for server -> hides server stalls (**Coordinated Omission!**).
     - `k6` Open-Loop (`executor: 'constant-arrival-rate', rate: 300, timeUnit: '1s'`): Fires 300 RPS relentlessly regardless of server slowness—exposing true queuing `P99`!
     - `TrafficShadowDiffer`: Mirror live read requests asynchronously (fire-and-forget with strict short timeout, never impacting user latency!) to `v2` handler and log response-body/status diffs (`16.2`).
   - **Verify:** Run closed-loop vs open-loop against a server with an artificial 1s GC pause every 10s -> record the dramatic `P99` difference!
3. **Daily Split:**
   - **DSA (1 hr):** LC 322 & LC 198 & LC 300 (DP speed drills).
   - **HLD (30 min):** Explain **Coordinated Omission** clearly and distinguish **Load**, **Stress** (find breaking point), **Soak** (catch memory/FD leaks over hours), and **Spike** tests. Also cover mutation testing (`16.2` `[T3]`).
   - **LLD (30 min):** Design an `AsyncDarkLaunchMirror` sampling 5% of production `GET` requests, stripping auth headers or running read-only against shadow v2, and emitting structured diff metrics.

---

### Day 136 — Tail-Latency Engineering (`P99`/`P999`), Tail-at-Scale & Little's Law (`17.3`)
1. **Pre-Requisite Learning Resource:**
   - Jeffrey Dean & Luiz André Barroso — *The Tail at Scale* (`cacm.acm.org/research/the-tail-at-scale/` — fan-out amplification, hedged requests) + Little's Law ($L = \lambda W$) & connection-pool sizing (*HikariCP Wiki: About Pool Sizing*!).
2. **Hands-On Activity:**
   - **What to build:** A **Tail-Latency & Pool-Sizing Lab** (`src/perf_lab/tail_lab.py`) demonstrating Little's Law, tail-at-scale fan-out amplification, and optimal DB connection pool tuning.
   - **Functions & Components:**
     - `simulate_fanout_tail(num_shards=50, single_node_p99_ms=100)`: Simulates 1 request fanning out to 50 shards in parallel (`max` latency!) -> prove empirically that `1 - 0.99^50 = 39.5%` of user requests hit the slow `P99` tail!
     - **Pool-Sizing Experiment:** Run 200 RPS against Postgres with `PgBouncer` pool size = `100` (context-switching & lock thrashing!) vs `12` (`~2 * cores + disk`) -> measure actual `P99` throughput improvement with the *smaller* pool!
     - **Hot-Path Allocation Elimination:** Profile and optimize your JSON serialization (`json` -> `orjson`) and connection reuse (`Keep-Alive`).
   - **Verify:** Apply Little's Law ($L = \lambda \times W$) to compute exact queue depth and concurrency required for 1,000 RPS at 20ms average latency ($L = 20$ concurrent requests!).
3. **Daily Split:**
   - **DSA (1 hr):** LC 42 & LC 11 & LC 84 (Array/Stack hard drills).
   - **HLD (30 min):** Why does adding *more* DB connections or *more* worker threads often **hurt** throughput and spike `P99` latency?
   - **LLD (30 min):** Design a `HedgedRequestExecutor` (`call_with_hedge(fn, hedge_delay_p95_ms)`): sends primary request, and if no reply after `P95` delay (e.g., 15ms), fires a 2nd idempotent backup request and takes whichever finishes first (capped by a 10% hedge budget!).

---

### Day 137 — Resilience: Load Shedding, Adaptive Concurrency, Circuit Breakers & Bulkheads (`17.2`)
1. **Pre-Requisite Learning Resource:**
   - Netflix TechBlog — *Performance Under Load: Adaptive Concurrency Limits* (Vegas / Gradient2 algorithms, Little's Law) + AWS Builders' Library (*Timeouts, Retries, and Backoff with Jitter* + *Avoiding Fallback in Distributed Systems*).
2. **Hands-On Activity:**
   - **What to build:** Implement a full **Resilience Middleware Stack** (`src/resilience/guards.py`) protecting your FastAPI service from downstream cascading failures.
   - **Functions & Components:**
     - `TenantBulkhead`: Separate `asyncio.Semaphore` per downstream dependency and per tenant tier so a slow Partner B endpoint cannot exhaust connections needed by healthy Partner A!
     - `RetryBudget`: Tracks `total_requests` vs `retry_requests` over a sliding 10s window; refuses retries once retries exceed `20%` of normal traffic (preventing **metastable retry storms**!).
     - `GradientConcurrencyLimiter`: Dynamically adjusts `max_inflight` limit using observed `min_rtt / current_rtt` (shrinking concurrency immediately when queue wait raises `current_rtt`!).
     - `DeadlinePropagator`: Enforces remaining timeout budget across `Service A -> Service B -> DB` so Service B never wastes work after Service A already timed out!
   - **Break Intentionally:** Slow downstream dependency to 3s under 200 RPS `k6` load: watch unprotected server collapse completely (`100% 500s`), then enable Bulkhead + Circuit Breaker + Retry Budget + Load Shedding (`503 Retry-After`) and watch healthy endpoints stay `99.9%` green!
3. **Daily Split:**
   - **DSA (1 hr):** LC 146 & LC 362 (Cache & sliding window counter).
   - **HLD (30 min):** Explain a **Metastable Failure** (how a temporary traffic spike triggers retries -> retries double load -> cache evicts -> latency climbs -> retries continue keeping the system wedged in collapse *even after* user traffic returns to normal!).
   - **LLD (30 min):** Implement `Gradient2ConcurrencyLimiter` (`on_acquire()`, `on_success(rtt_ms)`, `on_drop()`) maintaining EWMA short/long RTT baselines.

---

### Day 138 — Chaos Engineering GameDay (`16.2`)
1. **Pre-Requisite Learning Resource:**
   - Principles of Chaos Engineering (`principlesofchaos.org`) + Linux `man 8 tc-netem` (network packet loss, latency, jitter injection).
2. **Hands-On Activity:**
   - **What to build:** Execute a structured **Chaos GameDay** on your `k3s` / Docker stack on Ubuntu VM while `k6` drives steady 100 RPS load and Grafana/Jaeger record live telemetry:
     1. **Network Chaos:** Inject `tc qdisc add dev eth0 root netem delay 150ms 50ms loss 2%` on the DB/downstream container -> observe `P99` & retry budget behavior.
     2. **Pod Kill:** Delete 1 of 2 API pods in `k3s` mid-load -> verify `preStop` + `readinessProbe` + `PDB` yield zero dropped connections.
     3. **Dependency Outage:** Pause Redis container (`docker pause`) for 20s -> verify fail-open rate limiter + DB connection pool limits keep service alive!
     4. **Clean up:** Remove `tc netem` rules (`tc qdisc del ...`) cleanly afterward.
3. **Daily Split:**
   - **DSA (1 hr):** LC 207 & LC 684 & LC 743 (Graph speed drill).
   - **HLD (30 min):** How do you run Chaos Engineering safely in **production** without causing real customer outages (steady-state hypothesis, strict blast-radius containment, automated abort triggers)?
   - **LLD (30 min):** Design a `FaultInjectorMiddleware` configured by header/percentage (`latency_ms`, `abort_status`, `probability`) for staging chaos testing.

---

### Day 139 — SRE: SLIs, SLOs, Burn-Rate Alerts, Incident Triage & Disaster Recovery (`17.4`)
1. **Pre-Requisite Learning Resource:**
   - Google SRE Book — Chapter 4 (*Service Level Objectives*) & Google SRE Workbook Chapter 5 (*Alerting on SLOs*: Multi-Window Multi-Burn-Rate Alerts!) + Postmortem Culture (Ch. 15).
2. **Hands-On Activity:**
   - **What to build:** Write real **Prometheus Multi-Window Burn-Rate Alert Rules** (`k8s/slo_alerts.yaml`), run a **Postgres Backup & Restore DR Drill**, and write a **1-Page Blameless Postmortem** (`docs/gameday_postmortem.md`) for Day 138's outages!
   - **Functions & Components:**
     - SLO: `99.9%` availability over 30 days (error budget = `0.1%` = 43.2 min downtime/month).
     - Burn-Rate Rules: Page immediately if `14.4x` burn rate over `1h` AND `5m` (consumes 2% of monthly budget in 1h!); ticket alert if `3x` burn over `6h` + `30m`.
     - **DR Drill (`RPO`/`RTO`):** Take `pg_basebackup` / `pg_dump` -> drop a critical table (`DROP TABLE orders`) -> restore into a fresh Postgres instance -> measure exact **RTO (minutes to recover)** and **RPO (seconds of lost commits)**!
   - **Verify:** Audit alerts to verify every alert is **symptom-based** (user pain) with a linked runbook—not noisy cause-based alerts ("CPU 75%").
3. **Daily Split:**
   - **DSA (1 hr):** LC 23 & LC 253 & LC 56 (Heap/Interval speed drill).
   - **HLD (30 min):** Design a **Multi-Region Disaster Recovery Plan** (`RPO < 5s`, `RTO < 2 min`) covering automated backup verification drills, split-brain prevention on failover, and DNS/Anycast traffic shifting.
   - **LLD (30 min):** Design an `ErrorBudgetTracker` computing remaining error budget %, current burn rate over 5m/1h/6h windows, and deployment freeze flag (`freeze_deploys = budget <= 0`).

---

### Day 140 — Capacity Planning & Cloud FinOps (`17.4`)
1. **Pre-Requisite Learning Resource:**
   - FinOps Foundation Framework (*Inform, Optimize, Operate* + *Unit Economics*) + AWS/Azure Cloud Cost Optimization patterns (Cross-AZ transfer traps, NAT Gateway vs VPC Endpoints, Spot/Preemptible Spark executors, S3/Blob tiering).
2. **Hands-On Activity:**
   - **What to build:** Build a **Unit Economics & Capacity Model Spreadsheet/Script** (`src/perf_lab/finops_model.py`) calculating exact monthly cloud cost for a Series B/C platform handling 50M requests/day + 5 TB/day analytics ingestion.
   - **Functions & Components:**
     - `estimate_api_tier_cost(rps, avg_cpu_ms, mem_mb)`: Sizes pods/nodes with 40% N+2 headroom.
     - `estimate_network_trap_cost(tb_per_day)`: Quantifies hidden **Cross-AZ transfer ($0.01–0.02/GB each way!)** + **NAT Gateway processing ($0.045/GB)** vs **AZ-local topology routing + PrivateLink ($0)**!
     - `estimate_lakehouse_cost(tb_raw)`: Compares uncompressed JSON vs Zstd Parquet + Spot Spark executors + Azure Blob Hot/Cool/Archive lifecycle tiering.
   - **Verify:** Identify the top 3 cost-reduction levers (usually: Cross-AZ traffic, idle over-provisioned DBs/K8s nodes, uncompacted/un-tiered logs & S3 data).
3. **Daily Split:**
   - **DSA (1 hr):** LC 410 & LC 875 (Binary Search on Answer speed drill).
   - **HLD (30 min):** How do you design Kafka consumers, Spark jobs, and microservice calls in AWS/Azure to avoid paying massive **Cross-AZ network transfer bills** while preserving Multi-AZ high availability?
   - **LLD (30 min):** Design a `SpotInstanceGracefulPreemptor` catching cloud 2-minute spot-termination metadata notifications to checkpoint Spark/Flink state or drain K8s pods early.

---

### Day 141 — Probabilistic Data Structures From Scratch (`13.1`)
1. **Pre-Requisite Learning Resource:**
   - Flajolet et al. — *HyperLogLog* (conceptual intuition: leading zeros probability) + Cormode & Muthukrishnan — *Count-Min Sketch* + Fan et al. — *Cuckoo Filter: Practically Better Than Bloom* (`[T3]` vs Bloom) + Dunning — *t-digest* (`[T3]` quantile sketching).
2. **Hands-On Activity:**
   - **What to build:** Implement **Count-Min Sketch** and **HyperLogLog** from scratch in Python (`src/primitives/sketches.py`) and benchmark memory vs exact `Counter`/`set` on 2 million events!
   - **Functions & Components:**
     - `CountMinSketch(width, depth)`: `d` hash functions over a `d x w` 2D integer matrix; `add(item, count=1)` increments all `d` counters; `estimate(item) -> min(...)` (never underestimates, bounded overestimation!).
     - `HyperLogLog(p=12)`: Uses `m = 2^p = 4096` 6-bit registers (~3 KB RAM!); hashes item -> first `p` bits select register, remaining bits count `leading_zeros + 1`, computes harmonic mean with small/large-range bias correction!
   - **Verify:** Count 1,000,000 distinct items using HLL in **3 KB of RAM** and measure relative error (`~1.6%` vs `~80 MB` for a Python `set`!); detect Top-K heavy hitters with Count-Min Sketch.
3. **Daily Split:**
   - **DSA (1 hr):** LC 347 & LC 692 & LC 1642 (Frequency/Top-K Heap drills).
   - **HLD (30 min):** Where do Uber/Datadog/ClickHouse use **HyperLogLog** (unique riders per city/hour in real time), **Count-Min Sketch** (heavy-hitter hot-key detection in rate limiters/caches), **Bloom/Cuckoo Filters**, and **`t-digest`** (mergeable distributed `P99` percentiles)?
   - **LLD (30 min):** Design a `HeavyHitterDetector` combining a `CountMinSketch` + fixed-size Min-Heap of size $K$ to track streaming Top-$K$ hot keys in $O(1)$ memory!

---

### Day 142 — Geospatial Indexing: Geohash, QuadTree, S2 & Uber H3 (`13.1`)
1. **Pre-Requisite Learning Resource:**
   - Uber Engineering Blog — *H3: Uber's Hexagonal Hierarchical Spatial Index* (`uber.com/blog/h3/` — why hexagons beat squares: uniform neighbor distances, no corner distortions!) + QuadTree / Geohash boundary edge-case notes.
2. **Hands-On Activity:**
   - **What to build:** Build a **QuadTree** from scratch in Python AND benchmark **Uber `h3`** (`src/primitives/geo_index.py`) indexing 200k simulated drivers across a city!
   - **Functions & Components:**
     - `QuadTree(boundary_box, capacity=16)`: Recursively subdivides into `NW, NE, SW, SE` quadrants on overflow; `insert(driver_id, lat, lon)` and `query_range(circle_bbox) -> list`.
     - `H3DriverIndex(resolution=8)`: Converts `(lat, lon) -> h3_cell` (`h3.latlng_to_cell`), stores `dict[h3_cell, set[driver_id]]`, updates driver location in $O(1)$ (`remove` old cell, `add` new cell), and searches nearby drivers via `h3.grid_disk(user_cell, k=2)` (`k`-ring!).
   - **Verify:** Benchmark 1,000 proximity queries over 200k drivers: naive Haversine scan vs `QuadTree` vs `H3 grid_disk`!
3. **Daily Split:**
   - **DSA (1 hr):** LC 973 & LC 304 & QuadTree LC 427 *Construct Quad Tree*.
   - **HLD (30 min):** Why did Uber replace Geohash/S2 rectangles with **H3 Hexagons** for surge pricing and rider-driver matching? (Every 1-ring neighbor is equidistant; smooth gradients; hierarchical parent aggregation from res 9 -> res 7!).
   - **LLD (30 min):** Design an `InMemorySpatialDriverGrid` supporting 50,000 location updates/sec (`update_location(driver_id, lat, lon, ts)`) + stale-driver TTL expiry + `find_nearest_k(lat, lon, k)`.

---

### Day 143 — High-Throughput Ring Buffers & LMAX Disruptor Patterns (`1.1`, `13.2`)
1. **Pre-Requisite Learning Resource:**
   - Martin Thompson et al. — *LMAX Disruptor Technical Paper* (`lmax-exchange.github.io/disruptor/files/Disruptor-1.0.pdf` §1–4: why queues contend on locks/head-tail false sharing, ring buffers, sequence barriers) + *False Sharing & Cache-Line Locality* (`1.1`).
2. **Hands-On Activity:**
   - **What to build:** Implement a **Single-Producer Multi-Consumer Ring Buffer** (`src/primitives/ring_buffer.py`) with power-of-2 bitmask indexing (`idx & (cap - 1)`) and sequence-barrier tracking.
   - **Functions & Components:**
     - `RingBuffer(capacity_power_of_2=1024)`: Pre-allocates fixed slot list (zero GC allocation on hot path!).
     - `Sequencer`: Tracks `cursor` (published sequence) and `gating_sequences` (slowest consumer sequence so producer never overwrites unread slots!).
     - `ConsumerBarrier.wait_for(next_seq)`: Blocks/yields until `cursor >= next_seq`, then processes batch `[next_seq .. available]` in one pass!
   - **Verify:** Benchmark batch throughput vs Python `queue.Queue` and explain **CPU Cache-Line False Sharing** (why `head` and `tail` pointers sitting in the same 64-byte L1 cache line invalidate cores!).
3. **Daily Split:**
   - **DSA (1 hr):** LC 622 *Design Circular Queue* & LC 641 *Design Circular Deque* (fixed-array wraparound math).
   - **HLD (30 min):** Where do ring buffers / lock-free event pipelines sit in high-frequency trading, telemetry agents, and logging engines (`Log4j2` Disruptor)?
   - **LLD (30 min):** Design a `BatchingRingBufferLogger` accepting async log events without blocking caller threads, dropping or blocking cleanly per policy when full.

---

### Day 144 — Tier-1 Architecture #1 (Google / YouTube): Video Upload, Async Transcoding & CDN Streaming (`13.3`)
1. **Pre-Requisite Learning Resource:**
   - YouTube / Netflix TechBlog (*Adaptive Bitrate Streaming HLS/DASH*, *Chunked Encoding Pipelines*) + AWS Media/S3 Direct-Upload & CloudFront CDN Caching Architecture.
2. **Hands-On Activity:**
   - **What to build:** Design doc (`docs/tier1_designs/01_youtube_streaming.md`) + prototype a **DAG Chunked Media Processing Pipeline** (`src/tier1_labs/video_pipeline.py`) splitting input payloads into parallel segment jobs, transcoding concurrently, and merging an HLS-style manifest.
   - **Functions & Components:**
     - `ChunkUploader`: Splits large file into 5 MB chunks, issues presigned MinIO/Azure Blob URLs, tracks resumable upload state.
     - `TranscodeDAGOrchestrator`: Splits video into GOP segments, dispatches parallel transcode tasks (`360p`, `720p`, `1080p`) via bounded worker pool/Kafka, generates `.m3u8` adaptive playlist manifest, and pushes to object storage.
     - `BufferedViewCounter`: Batches millions of view pings in Redis/Kafka and flushes increments to Postgres/ClickHouse every 10s (avoiding row-lock contention on viral videos!).
   - **Verify:** Test resumable chunk recovery if 1 transcode worker fails mid-video without restarting already-completed segments!
3. **Daily Split:**
   - **DSA (1 hr):** LC 295 & LC 743 (Heap/Graph speed drills).
   - **HLD (30 min):** Complete whiteboard walkthrough of **YouTube**: Resumable Upload -> Object Storage -> Kafka -> DAG Transcode Workers (Spot instances!) -> Adaptive Bitrate HLS/DASH Segments -> CDN Edge Cache (hot vs long-tail storage tiering) + Sharded View Counter.
   - **LLD (30 min):** Design a `ViralCounterAggregator` sharding hot video counters across `N` striped buckets in Redis/memory to eliminate single-row lock contention.

---

### Day 145 — Tier-1 Architecture #2 (Amazon / Stripe): High-Throughput Financial Ledger & Order/Payment Platform (`13.3`)
1. **Pre-Requisite Learning Resource:**
   - Square / Modern Treasury Engineering Blog (*Double-Entry Bookkeeping in Software Engineering*) + Stripe Engineering Blog (*Idempotency & Payment Reconciliation*).
2. **Hands-On Activity:**
   - **What to build:** Design doc (`docs/tier1_designs/02_financial_ledger.md`) + pure-Postgres **Double-Entry Immutable Ledger** (`src/tier1_labs/double_entry.py`).
   - **Functions & Components:**
     - Tables: `accounts(id, currency, type)`, `journal_entries(id, idempotency_key UNIQUE, description, created_at)`, `postings(id, entry_id, account_id, amount_signed_cents)` — **ZERO `UPDATE`/`DELETE` allowed on `postings`!** Corrections happen ONLY via compensating reversal entries!
     - `post_balanced_entry(postings, idempotency_key)`: Enforces `sum(amounts) == 0` (every debit has an equal credit!) inside `SERIALIZABLE` / canonical-ordered `SELECT ... FOR UPDATE` on involved accounts.
     - `reconcile_external_psp_settlement()`: Compares internal ledger cleared postings against mock Stripe/PSP settlement CSV to flag discrepancies (`missing_in_psp`, `amount_mismatch`).
   - **Verify:** Prove across 10k concurrent transfers that the global sum of all `postings` across all accounts equals `0` to the exact cent!
3. **Daily Split:**
   - **DSA (1 hr):** LC 465 *Optimal Account Balancing* (debt settlement backtracking) & LC 1 (Two Sum).
   - **HLD (30 min):** Complete whiteboard walkthrough of **Amazon / Stripe Payment & Ledger Platform**: API Idempotency -> Payment Orchestrator (Temporal Saga!) -> External PSP -> Immutable Double-Entry Ledger DB (Sharded CockroachDB/Postgres) -> Transactional Outbox -> Kafka -> Iceberg Reconciliation Pipeline.
   - **LLD (30 min):** Design `LedgerPostingValidator` enforcing balanced currency pairs, non-negative asset/wallet constraints where applicable, and reversal-entry generation.

---

### Day 146 — Tier-1 Architecture #3 (LinkedIn / Datadog): Large-Scale Telemetry Ingestion & Activity Streams (`13.3`)
1. **Pre-Requisite Learning Resource:**
   - LinkedIn Engineering Blog (*FollowFeed / Economic Graph* & *Apache Pinot / Kafka at LinkedIn*) + Datadog / Uber M3 Telemetry Ingestion architecture (edge buffering, cardinality control, ClickHouse/tiered storage).
2. **Hands-On Activity:**
   - **What to build:** Design doc (`docs/tier1_designs/03_telemetry_and_feed.md`) + prototype a **Log/Metric Edge Collector & Cardinality Limiter** (`src/tier1_labs/telemetry_collector.py`).
   - **Functions & Components:**
     - `EdgeAgentRingBuffer`: Buffers incoming spans/logs in bounded memory, compresses (`zstd`) batches every 1s or 1 MB, drops lowest-priority `DEBUG` logs first under backpressure (**priority load shedding!**) so `ERROR`/traces survive!
     - `CardinalityGuard`: Uses Day 141's `HyperLogLog` per metric name to detect runaway label explosions (`unique_series > 10_000`) and automatically strips offending dynamic tags (`request_id`/`ip`) before ingesting into TSDB/ClickHouse!
   - **Verify:** Feed 100k metrics with an accidental `user_id` tag -> confirm `CardinalityGuard` catches and drops the toxic label automatically!
3. **Daily Split:**
   - **DSA (1 hr):** LC 359 & LC 362 & LC 1268.
   - **HLD (30 min):** Complete whiteboard walkthrough of **Datadog Telemetry Ingestion** (Edge Agents -> L7 Collector -> Kafka -> Flink Downsampling -> ClickHouse Hot + Iceberg Cold) AND compare briefly against **LinkedIn Connections / 2nd-Degree Network Graph** (Graph partitioning + Redis adjacency cache + BFS).
   - **LLD (30 min):** Design a `DownsamplingRollupAggregator` (`10s raw -> 1m min/max/sum/count/t-digest`) and `DegreeConnectionsBFS` stopping cleanly at hop 2/3.

---

### Day 147 — Tier-1 Architecture #4 (Buffer / Remote SaaS + Feature Store): Scheduled Publishing Engine & ML Parity (`13.3`)
1. **Pre-Requisite Learning Resource:**
   - Buffer Engineering / SaaS Multi-Tenant Architecture posts (scheduled social/webhook publishing, OAuth token lifecycle, tenant rate limits) + Feast / Tecton Docs (*Streaming–Batch Feature Parity & Point-in-Time Joins*).
2. **Hands-On Activity:**
   - **What to build:** Design doc (`docs/tier1_designs/04_scheduled_saas_and_feature_store.md`) + prototype BOTH:
     1. A **Buffer-Style Multi-Tenant Scheduled Post Dispatcher** (`src/tier1_labs/scheduled_publisher.py`) using a Hierarchical Timing Wheel / Postgres+Redis sorted-set poller, per-tenant OAuth token refresh locking (preventing 10 concurrent workers from invalidating a single rotating OAuth refresh token!), and external API rate-limit respect (`429 Retry-After`).
     2. An **ASOF Point-in-Time Join** in DuckDB/Python (`src/tier1_labs/pit_join.py`) preventing ML feature leakage!
   - **Verify:** Simulate 5 workers publishing simultaneously for 1 user with an expired OAuth token -> prove singleflight token refresh happens **once** while all 5 posts succeed!
3. **Daily Split:**
   - **DSA (1 hr):** LC 981 (Time-Based KV Store) & LC 1146 (Snapshot Array).
   - **HLD (30 min):** Complete whiteboard walkthrough of **Buffer Scheduled Publishing Platform** (10M scheduled posts firing at `:00` top-of-the-hour spikes! Timing wheels/sharded time-buckets + fair-share tenant queuing + OAuth token vault + dead-letter alerts) + **Real-Time ML Feature Store** parity.
   - **LLD (30 min):** Design a `SingleflightOAuthTokenRefresher` ensuring only 1 worker refreshes an expired tenant token while sibling workers await the refreshed credential.

---

### Day 148 — Tier-1 Architectures #5 & #6 (Google / Amazon): Multi-Tenant Job Scheduler & Multi-Region Cells (`13.3`)
1. **Pre-Requisite Learning Resource:**
   - Google Borg Paper (*Large-scale cluster management at Google with Borg* §2–3: quotas, preemption, admission control) + AWS Cell-Based Architecture (`docs.aws.amazon.com`).
2. **Hands-On Activity:**
   - **What to build:** Design docs (`docs/tier1_designs/05_job_scheduler.md` & `06_multi_region_cells.md`) + prototype a **Multi-Tenant Weighted Fair-Share & Preemption Scheduler** (`src/tier1_labs/fair_scheduler.py`).
   - **Functions & Components:**
     - `ClusterResourceManager`: Tracks tenant quotas `(guaranteed_slots, max_burst_slots)` and current usage.
     - `admit_or_preempt(job)`: Admits high-priority production SLA jobs immediately if cluster is full by **preempting** low-priority burst/backfill jobs running above their guaranteed quota!
     - `CellArchitectureChecklist`: Document cell sizing math, stateless global router, regional data pinning, and automated region evacuation drills (`RPO`/`RTO`).
3. **Daily Split:**
   - **DSA (1 hr):** LC 621 & LC 1851 & LC 253.
   - **HLD (30 min):** Complete whiteboard walkthrough of BOTH: (1) **Multi-Tenant Distributed Spark/Trino Query & Job Routing Platform** (admission control, queues, query routing, noisy-neighbor preemption) and (2) **Global Multi-Region Cell-Based Cloud Architecture** (Amazon-style blast-radius isolation).
   - **LLD (30 min):** Design a `DominantResourceFairnessCalculator` (DRF) balancing multi-dimensional `(CPU, RAM)` fairness across competing tenants.

---

### Day 149 — Full Tier-1 Mock Interview Loop #1 (Google / Amazon / 50 LPA+ Startup Simulation)
1. **Pre-Requisite Learning Resource:**
   - Zero new reading today—rely strictly on what is in your head!
2. **Hands-On Activity (Strict Timed Simulation):**
   - **Round 1 — DSA Coding (60 min, blank editor, zero AI/autocomplete):** Solve 2 unseen medium/hard problems: 1 Graph/Heap (e.g., LC 778 or LC 1353) + 1 Sliding Window/Design (e.g., LC 1438 or LC 355). Talk through edge cases and Big-O aloud.
   - **Round 2 — High-Level System Design (45 min, blank page):** Pick **YouTube Video Streaming Pipeline** or **Amazon/Stripe Financial Ledger**—do full requirements, QPS/storage math, APIs, storage/partitioning, failure modes, and observability.
   - **Round 3 — LLD + Concurrency (45 min, blank editor):** Code a **Thread-Safe In-Memory Pub/Sub Broker with Topic Wildcards & Per-Subscriber Bounded Backpressure Queues** (`publish`, `subscribe`, `unsubscribe`, `shutdown`).
   - **Round 4 — Architecture Deep Dive (30 min):** Defend your Month 4 CDC -> Kafka -> Flink -> Iceberg/ClickHouse pipeline against ruthless "what breaks if X dies?" probing.
3. **Daily Split:**
   - *(Today's full mock loop replaces the standard 1h/30m/30m split—spend the remaining 30 min grading every weak spot brutally in `notes.md`.)*

---

### Day 150 — Full Curriculum Audit, Mock Loop #2 & Cloud Cleanup
1. **Pre-Requisite Learning Resource:**
   - Open [Curriculum.md](./Curriculum.md) from Section `1.1` through Section `17.4` line by line.
2. **Hands-On Activity:**
   - **Step 1 — Full [Curriculum.md](./Curriculum.md) Audit:** Check every single subconcept across all 17 sections (`1.1` to `17.4`). Verify you can explain/implement `[T1]`/`[T2]` topics with real terminal evidence and `[T3]` topics cleanly from memory. Flag any remaining weak spots for targeted weekend polish.
   - **Step 2 — Mock Loop #2 (Targeted Weakness Loop):**
     - **DSA (1 hr):** Re-solve the hardest pattern you stumbled on yesterday + 1 fresh Graph/Heap problem.
     - **HLD (30 min):** Timed blank-page design of **LinkedIn Activity Feed + Search** OR **Buffer Multi-Tenant Scheduled Publishing Platform** (Kafka, timing wheels/Temporal, Postgres, Redis, rate limits, observability).
     - **LLD (30 min):** Timed blank-page implementation of **Distributed-Style Token Bucket + Circuit Breaker Middleware**.
   - **Step 3 — Clean Shutdown & $0 Azure Audit:** Verify `terraform destroy` removed all non-free Azure resources, confirm Azure Cost Management shows `$0.00` (or within free tier), and clean up temporary test containers on your Ubuntu 16 GB VM.
