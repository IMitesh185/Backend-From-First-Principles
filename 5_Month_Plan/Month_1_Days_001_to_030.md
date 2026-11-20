# Month 1 (Days 1–30): Python OOP Beyond `for`-Loops, Linux/Networking, `localhost` -> Public HTTPS & Docker

> **Curriculum Coverage:** `1.1`, `1.5`, `2.1`, `2.2` (intro), `2.3–2.6`, `3.4`, `3.5`, `4.1–4.3`, `4.5`, `13.2` (LLD), `15.1`, `15.3` (CI/CD), `16.1` (unit tests)
> **Environment:** Main Laptop + Ubuntu 16 GB VM (SSH) + Azure Free Tier `B1s` VM (starting Day 17)

---

### Day 1 — Python Project Structure, Packaging & Static Tooling (`3.5`)
1. **Pre-Requisite Learning Resource:**
   - Python Packaging User Guide: *Writing your `pyproject.toml`* (`packaging.python.org/en/latest/guides/writing-pyproject-toml/`) + Astral `uv` Docs (*Projects & Lockfiles*) + `ruff` & `mypy` Getting Started docs.
2. **Hands-On Activity:**
   - **What to build:** Set up SSH key auth to your Ubuntu 16 GB VM from your main laptop (`~/.ssh/config`), install Python 3.12 + `uv`, and scaffold a reproducible backend project skeleton (`src/` layout, `tests/`, `pyproject.toml`, `uv.lock`).
   - **Functions & Components:**
     - `pyproject.toml`: Must declare project metadata, runtime vs dev dependency groups (`pytest`, `mypy`, `ruff`), `[tool.ruff]` strict lint rules, and `[tool.mypy]` `strict = true`.
     - `src/app/health.py` -> `check_environment() -> dict[str, str]`: Reads Python version, OS platform, and `APP_ENV` env var; raises `RuntimeError` if `APP_ENV` is unset.
     - `tests/test_health.py` -> `test_check_environment_returns_metadata(monkeypatch)` & `test_missing_env_raises(monkeypatch)`: Verifies both paths cleanly.
   - **Verify:** Run `uv run ruff check .`, `uv run mypy src`, and `uv run pytest` over SSH on the Ubuntu VM.
3. **Daily Split:**
   - **DSA (1 hr):** LC 217 *Contains Duplicate* & LC 242 *Valid Anagram* — solve with `set` and `dict` frequency maps ($O(n)$ time, no sorting).
   - **HLD (30 min):** Sketch on blank paper what happens on a single server from `git push` on your laptop to running a Python process on a remote Linux machine (list every unknown box honestly).
   - **LLD (30 min):** Design a `ConfigLoader` class specification: fields (`app_env`, `port`, `log_level`), validation rules in `__post_init__`, and immutability (`frozen=True`).

---

### Day 2 — Python Data Model, State Encapsulation & Invariants (`3.4`)
1. **Pre-Requisite Learning Resource:**
   - *Fluent Python* (2nd Ed., Luciano Ramalho) — Chapter 1 (*The Python Data Model*: `__repr__`, `__eq__`, `__hash__`) & Chapter 11 (*A Pythonic Object*: slots, properties, private attributes).
2. **Hands-On Activity:**
   - **What to build:** A pure-Python domain model module (`src/domain/ledger_entry.py`) enforcing strict financial invariants without any database or web framework.
   - **Functions & Components:**
     - `InvalidAmountError(Exception)` & `CurrencyMismatchError(Exception)`: Domain-specific exception classes.
     - `Money` dataclass (`frozen=True`, `slots=True`): Contains `amount_minor_units: int` (never `float`) and `currency: str`; `__post_init__` rejects negative amounts or non-3-letter ISO currency codes; implements `__add__` rejecting mismatched currencies.
     - `Account` class: Holds `account_id`, `_balance: Money` (protected), `credit(money: Money) -> None`, `debit(money: Money) -> None` (raises `InsufficientFundsError` if balance would go negative), and read-only `@property def balance`.
   - **Verify & Break:** Write unit tests attempting negative money, float inputs, currency mismatch, and overdraft; confirm every illegal state fails loudly.
3. **Daily Split:**
   - **DSA (1 hr):** LC 1 *Two Sum* & LC 49 *Group Anagrams* — master hash-map complement lookup and canonical tuple keys (`O(n)` / `O(n * k)`).
   - **HLD (30 min):** Sketch why floating-point numbers corrupt financial systems and how integer minor-units + immutable ledger entries fit into a simple payment API (`Client -> API -> DB`).
   - **LLD (30 min):** Design `ShortURL` domain entity: attributes (`code`, `long_url`, `created_at`, `expires_at`, `click_count`), state-transition methods (`record_click()`, `is_expired(now)`), and invariants.

---

### Day 3 — Composition Over Inheritance & Structural Interfaces (`Protocol` / `ABC`) (`3.4`)
1. **Pre-Requisite Learning Resource:**
   - *Fluent Python* (2nd Ed.) — Chapter 13 (*Interfaces, Protocols, and ABCs*: Duck Typing, `typing.Protocol`, `abc.ABC`, Goose Typing) + PEP 544 (*Protocols: Structural subtyping*).
2. **Hands-On Activity:**
   - **What to build:** Build the repository abstraction and Base62 encoder for an In-Memory URL Shortener in pure Python (zero FastAPI, zero DB).
   - **Functions & Components:**
     - `Clock(Protocol)`: Defines `now() -> datetime`. Concrete implementations: `SystemClock` and `FakeClock(start_time)` (with `advance(seconds: int)`).
     - `URLRepository(Protocol)`: Defines `save(short_url: ShortURL) -> None`, `get_by_code(code: str) -> ShortURL | None`, `increment_clicks(code: str) -> None`.
     - `InMemoryURLRepository`: Backed by an internal `dict[str, ShortURL]`; enforces uniqueness on `save` (raises `DuplicateCodeError`).
     - `Base62Encoder`: `encode(num: int) -> str` and `decode(code: str) -> int` using `[0-9a-zA-Z]` alphabet; validates non-negative ints and invalid characters.
   - **Verify:** Write `pytest` round-trip tests for `Base62Encoder` (`0`, `61`, `62`, `10**12`) and `InMemoryURLRepository` duplicate rejection.
3. **Daily Split:**
   - **DSA (1 hr):** LC 347 *Top K Frequent Elements* (bucket sort $O(n)$ approach + compare vs heap) & LC 128 *Longest Consecutive Sequence* ($O(n)$ HashSet).
   - **HLD (30 min):** Draw how swapping `InMemoryURLRepository` for `PostgresURLRepository` keeps business logic untouched (Hexagonal / Ports & Adapters boundary).
   - **LLD (30 min):** Design `ShortenerService` class accepting `(repo: URLRepository, clock: Clock, id_gen: IdGenerator)` via constructor injection: define `shorten(long_url, ttl_seconds)` and `resolve(code)`.

---

### Day 4 — Pure-Python LLD Capstone: URL Shortener & Test Doubles (`3.4`, `16.1`)
1. **Pre-Requisite Learning Resource:**
   - Martin Fowler — *Mocks Aren't Stubs* (`martinfowler.com/articles/mocksArentStubs.html`) + `pytest` Docs (*Fixtures & Monkeypatch*).
2. **Hands-On Activity:**
   - **What to build:** Complete the pure-Python `ShortenerService` and test it deterministically using fakes (`FakeClock`, `InMemoryURLRepository`, `FailingURLRepository`).
   - **Functions & Components:**
     - `SequentialIdGenerator`: Thread-unsafe counter `next_id() -> int` starting at `100_000` (so codes start multi-char).
     - `ShortenerService`:
       - `shorten(long_url: str, ttl_seconds: int | None = None) -> ShortURL`: Validates `http://`/`https://` scheme via `urllib.parse.urlparse`, generates ID, encodes Base62, sets `expires_at` using `clock.now()`, saves via `repo`.
       - `resolve(code: str) -> str`: Looks up `code` (raises `CodeNotFoundError`), checks `is_expired(clock.now())` (raises `CodeExpiredError`), calls `repo.increment_clicks(code)`, returns `long_url`.
     - `FailingURLRepository` (test double): Always raises `StorageUnavailableError` on `save` to test error handling.
   - **Verify:** Test expiry using `FakeClock.advance(3601)` without `time.sleep`, invalid URL schemes, and storage error propagation.
3. **Daily Split:**
   - **DSA (1 hr):** LC 125 *Valid Palindrome* & LC 167 *Two Sum II* & LC 15 *3Sum* — two-pointer convergence invariants.
   - **HLD (30 min):** URL Shortener v1 (Single-Node): 2 endpoints (`POST /shorten`, `GET /{code}`), 1 table, Base62 math ($62^7 \approx 3.5T$), read-vs-write QPS assumptions.
   - **LLD (30 min):** Design a `SlidingWindowCounter` or `TokenBucket` rate-limiter class with `Clock` injection (`allow_request(client_id: str) -> RateLimitDecision`).

---

### Day 5 — Pure-Python LLD Build: Token Bucket Rate Limiter & Context Managers (`3.4`, `13.1`, `13.2`)
1. **Pre-Requisite Learning Resource:**
   - Stripe Engineering Blog — *Scaling your API with rate limiters* (`stripe.com/blog/rate-limiters`) + Python Docs (*`contextlib` & Context Manager Protocol `__enter__`/`__exit__`*).
2. **Hands-On Activity:**
   - **What to build:** A pure-Python `TokenBucketRateLimiter` module (`src/ratelimit/token_bucket.py`) + a timing/resource `contextmanager`.
   - **Functions & Components:**
     - `RateLimitDecision` dataclass: `allowed: bool`, `remaining_tokens: int`, `retry_after_seconds: float`.
     - `BucketState` dataclass: `tokens: float`, `last_refill_ts: float`.
     - `TokenBucketRateLimiter`: Init takes `(capacity: int, refill_rate_per_sec: float, clock: Clock)`; `allow(key: str, cost: int = 1) -> RateLimitDecision` computes elapsed time since `last_refill_ts`, adds `elapsed * refill_rate_per_sec` capped at `capacity`, deducts `cost` if sufficient, else computes exact `retry_after_seconds`.
     - `timed_operation(name: str)` context manager: Measures `time.perf_counter()` duration and guarantees logging even when an exception is raised.
   - **Verify:** Test burst capacity exhaustion, fractional refill over `FakeClock.advance(0.5)`, and exact `retry_after_seconds` math.
3. **Daily Split:**
   - **DSA (1 hr):** LC 11 *Container With Most Water* & LC 42 *Trapping Rain Water* — two-pointer max-height invariant.
   - **HLD (30 min):** Where does a rate limiter sit (`API Gateway / Reverse Proxy` vs `App Middleware`), and why does an in-memory per-process bucket fail behind 4 API instances?
   - **LLD (30 min):** Design an `LRUCache(capacity: int)` from scratch using a `dict[K, Node]` and an explicit `DoublyLinkedList` (no `OrderedDict`).

---

### Day 6 — Pure-Python LLD Build: LRU Cache From Scratch (`13.2`)
1. **Pre-Requisite Learning Resource:**
   - *Design Patterns / Data Structures*: Python Docs (`collections.OrderedDict` source/behavior just to understand what you're replacing) + *Database Internals* Ch. 5 brief section on Page Cache eviction (*LRU vs Clock*).
2. **Hands-On Activity:**
   - **What to build:** A generic typed `LRUCache[K, V]` backed by a sentinel-head/sentinel-tail doubly linked list (`src/cache/lru.py`) with hit/miss metrics.
   - **Functions & Components:**
     - `_Node[K, V]`: Stores `key`, `value`, `prev`, `next`.
     - `LRUCache[K, V]`:
       - `_remove(node)`: Unlinks `node` in $O(1)$.
       - `_insert_at_front(node)`: Inserts right after dummy `head` in $O(1)$.
       - `get(key: K) -> V | None`: Moves node to front on hit, increments `hits`/`misses`, returns value.
       - `put(key: K, value: V) -> None`: Updates existing node & moves to front, or inserts new node and evicts `tail.prev` if `len > capacity` (deleting evicted key from `dict`).
       - `stats() -> CacheStats`: Returns hits, misses, hit ratio, current size.
   - **Verify & Break:** Test `capacity=0`, `capacity=1`, updating an existing key when full (must NOT evict another key), and verify $O(1)$ behavior on 100k operations.
3. **Daily Split:**
   - **DSA (1 hr):** LC 146 *LRU Cache* (code from memory without notes) + LC 3 *Longest Substring Without Repeating Characters* (sliding window).
   - **HLD (30 min):** Design an in-process L1 cache in front of a URL Shortener DB: calculate memory needed for 10M cached `(short_code, long_url)` pairs.
   - **LLD (30 min):** Extend `LRUCache` on paper to `TTLLRUCache`: how does adding `expires_at` affect `get()`, `put()`, and expired-entry cleanup?

---

### Day 7 — Decorators, `functools.wraps`, Closures & Retry Utility (`3.4`, `17.2`)
1. **Pre-Requisite Learning Resource:**
   - *Fluent Python* (2nd Ed.) — Chapter 9 (*Decorators and Closures*: `nonlocal`, `functools.wraps`, parameterized decorators) + AWS Architecture Blog — *Exponential Backoff and Jitter* (`aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/`).
2. **Hands-On Activity:**
   - **What to build:** A reusable `@retry` decorator (`src/resilience/retry.py`) supporting transient exception filtering, max attempts, exponential backoff, and **Full Jitter**.
   - **Functions & Components:**
     - `compute_backoff_delay(attempt: int, base_delay: float, max_delay: float, rng: random.Random) -> float`: Computes `min(max_delay, base_delay * 2**attempt)` then applies Full Jitter `rng.uniform(0, cap)`.
     - `retry(exceptions: tuple[type[Exception], ...], max_attempts: int, base_delay: float, max_delay: float, sleep_fn: Callable[[float], None])`: Preserves wrapped function metadata via `@functools.wraps`; retries only specified transient exceptions; re-raises immediately on non-transient exceptions or after `max_attempts`.
   - **Verify:** Inject a mock `sleep_fn` and a flaky callable that fails twice with `TimeoutError` then succeeds; assert exact retry count and non-retry on `ValueError`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 424 *Longest Repeating Character Replacement* & LC 567 *Permutation in String* — fixed/dynamic sliding-window counts.
   - **HLD (30 min):** Draw what happens when 10,000 clients retry simultaneously without jitter (**Thundering Herd / Retry Storm**) vs Full Jitter + Retry Budget.
   - **LLD (30 min):** Design a `CircuitBreaker` class (`CLOSED`, `OPEN`, `HALF_OPEN`) with failure threshold, recovery timeout, and `call(fn)` wrapper.

---

### Day 8 — Linux Processes, Syscalls, Signals & `/proc` Inspection (`1.1`, `1.5`)
1. **Pre-Requisite Learning Resource:**
   - *Operating Systems: Three Easy Pieces (OSTEP)* — Chapter 4 (*The Abstraction: The Process*) & Chapter 5 (*Interlude: Process API — `fork`, `exec`, `wait`*) + Linux `man 7 signal` & `man 5 proc`.
2. **Hands-On Activity:**
   - **What to build:** On your Ubuntu 16 GB VM, write small Python process/signal inspection scripts (`src/os_lab/proc_lab.py`) and inspect kernel state directly.
   - **Functions & Components:**
     - `spawn_and_create_zombie()`: Uses `os.fork()`; child exits immediately while parent sleeps 15s without calling `os.waitpid()`, letting you observe `Z` (defunct) state in `ps aux`.
     - `signal_graceful_worker()`: Registers `signal.signal(SIGTERM, handler)` and `SIGINT`; prints PID, sets a `shutdown_requested = True` flag (safe signal handling), finishes current simulated work loop, and cleans up a temp PID file before exiting `0`.
     - `inspect_self_proc()`: Reads `/proc/self/status`, `/proc/self/fd`, and `/proc/self/limits` directly to print open file descriptors and soft/hard `RLIMIT_NOFILE`.
   - **Verify & Break:** Run `strace -e trace=process,signal python3 ...` to watch `clone`/`fork`, `execve`, `rt_sigaction`, and `wait4` syscalls live; send `SIGTERM` vs `SIGKILL` (`kill -9`).
3. **Daily Split:**
   - **DSA (1 hr):** LC 76 *Minimum Window Substring* (hard sliding window—master `need`/`have` counter logic).
   - **HLD (30 min):** Explain how a multi-worker web server (`gunicorn` master + 4 workers) uses `fork()` and `SIGTERM`/`SIGHUP` for zero-downtime reloads.
   - **LLD (30 min):** Design a `GracefulShutdownCoordinator` class managing active request counter, `is_draining` state, and bounded wait timeout.

---

### Day 9 — Network Stack: IPv4/CIDR, Routing, DNS & Socket Listen Queues (`2.1`)
1. **Pre-Requisite Learning Resource:**
   - Cloudflare Blog — *SYN packet handling in the Linux kernel* (`blog.cloudflare.com/syn-packet-handling-in-the-linux-kernel/`) + RFC 1034 §3–4 (DNS concepts) + Linux `man 2 listen` & `man 7 tcp`.
2. **Hands-On Activity:**
   - **What to build:** A CLI CIDR & DNS diagnostic script (`src/net_lab/net_inspect.py`) + kernel socket queue inspection on Ubuntu VM.
   - **Functions & Components:**
     - `subnet_summary(cidr: str) -> dict`: Uses `ipaddress.ip_network` to compute network address, broadcast, usable host range, prefix length, and whether two CIDRs overlap (VPC/subnet conflict check!).
     - `trace_dns_records(domain: str)`: Queries `A`, `AAAA`, `CNAME`, `TXT` via `dnspython` (or subprocess `dig`), printing TTLs and authoritative answers.
     - **Kernel Queue Probe:** Inspect `/proc/sys/net/core/somaxconn`, `/proc/sys/net/ipv4/tcp_max_syn_backlog`, and `ss -lnt` (`Recv-Q` vs `Send-Q` on listening sockets).
   - **Verify:** Run `dig +trace google.com` and `ip route get 8.8.8.8` on Ubuntu VM; compute CIDR splits for a `/16` VPC divided into public/private `/24` subnets per AZ.
3. **Daily Split:**
   - **DSA (1 hr):** LC 20 *Valid Parentheses* & LC 155 *Min Stack* (auxiliary min tracking in $O(1)$).
   - **HLD (30 min):** Design VPC subnet CIDR allocation for a 3-AZ startup cloud environment (public LB subnets, private app subnets, private DB subnets—avoid overlapping ranges).
   - **LLD (30 min):** Design a `DNSCache` class storing records with `expires_at` (TTL) and negative-caching TTL for `NXDOMAIN` responses.

---

### Day 10 — Raw TCP Socket Server & HTTP/1.1 Framing (`2.1`, `2.3`)
1. **Pre-Requisite Learning Resource:**
   - *Beej's Guide to Network Programming* — Section 5 (*System Calls or Bust*: `socket`, `bind`, `listen`, `accept`, `send`, `recv`) + RFC 9112 (*HTTP/1.1 Message Syntax & Routing*: §2 Message, §6 Message Body Length).
2. **Hands-On Activity:**
   - **What to build:** A raw TCP HTTP/1.1 server in pure Python (`src/raw_http/server.py`) with zero HTTP libraries (`http.server` banned).
   - **Functions & Components:**
     - `create_listening_socket(host: str, port: int, backlog: int) -> socket.socket`: Sets `SO_REUSEADDR`, binds, listens.
     - `read_http_request(conn: socket.socket) -> HttpRequest | None`: Loops `conn.recv(4096)` into a buffer until `\r\n\r\n` is found (handles partial TCP reads!), parses request line (`method`, `path`, `version`) + headers case-insensitively, and reads exact `Content-Length` body bytes if present.
     - `format_http_response(status: int, reason: str, headers: dict[str, str], body: bytes) -> bytes`: Computes `Content-Length`, serializes valid `\r\n`-delimited headers + body.
     - `serve_forever(host: str, port: int)`: Accepts connections and handles `GET /health`, `POST /echo`, and `404` / `400`.
   - **Verify & Break:** Hit from your main laptop with `curl -v http://<ubuntu-vm-ip>:8080/echo -d '{"hi":1}'`. Break it by temporarily omitting `Content-Length` without closing the socket and watch `curl` hang.
3. **Daily Split:**
   - **DSA (1 hr):** LC 150 *Evaluate Reverse Polish Notation* & LC 739 *Daily Temperatures* (monotonic decreasing stack).
   - **HLD (30 min):** Draw why a single-threaded blocking `accept()` loop stalls when Client 1 opens a TCP connection and sends 1 byte per second (**Slowloris attack**).
   - **LLD (30 min):** Design an incremental `HttpRequestParser` state machine (`READING_REQUEST_LINE`, `READING_HEADERS`, `READING_BODY`, `COMPLETE`) handling partial chunk arrivals.

---

### Day 11 — HTTP/1.1 Keep-Alive, Pipelining Limits, HTTP/2 & QUIC (`2.2`, `2.3`)
1. **Pre-Requisite Learning Resource:**
   - *High Performance Browser Networking (HPBN)* by Ilya Grigorik (`hpbn.co`) — Chapter 2 (*Building Blocks of TCP*: 3-way handshake, slow start, flow control, Nagle) & Chapter 12 (*HTTP/2*: binary framing, multiplexing, HPACK, HOL blocking) + RFC 9000 §1–2 (QUIC overview).
2. **Hands-On Activity:**
   - **What to build:** Upgrade Day 10's raw HTTP/1.1 server to support **`Keep-Alive` persistent connections**, idle timeouts, and `Connection: close`.
   - **Functions & Components:**
     - `handle_connection(conn: socket.socket, idle_timeout: float)`: Sets `conn.settimeout(idle_timeout)`, loops calling `read_http_request(conn)` over the *same* TCP socket until client sends `Connection: close`, EOF occurs, or `socket.timeout` triggers.
     - `set_tcp_nodelay(conn: socket.socket)`: Toggles `TCP_NODELAY` (disabling Nagle's algorithm) and measures latency on small sequential writes.
   - **Verify:** Send 5 sequential requests in one `curl -v http://<ubuntu-vm-ip>:8080/health http://<ubuntu-vm-ip>:8080/health ...` call; verify `curl` prints `Re-using existing connection` and `ss -tan` shows 1 TCP connection instead of 5 `TIME_WAIT` sockets.
3. **Daily Split:**
   - **DSA (1 hr):** LC 84 *Largest Rectangle in Histogram* (monotonic stack mastery).
   - **HLD (30 min):** Compare HTTP/1.1 (6 TCP connections per browser origin) vs HTTP/2 (1 multiplexed TCP connection -> TCP HOL blocking on packet loss) vs HTTP/3 over QUIC (independent UDP streams + connection migration on WiFi -> 4G switch).
   - **LLD (30 min):** Design a `ConnectionPool` interface (`acquire()`, `release()`, `close_expired()`) tracking idle vs active connections and max pool size.

---

### Day 12 — Layered Web Service Architecture: Router, Service, Repository & DTOs (`4.1`, `4.2`)
1. **Pre-Requisite Learning Resource:**
   - FastAPI Official Docs (*Tutorial: Bigger Applications - Multiple Files*, *Dependencies*, *Response Model*) + Pydantic v2 Docs (*Models & Validation*).
2. **Hands-On Activity:**
   - **What to build:** Build the production-structured FastAPI service wrapping your Stage 1 URL Shortener (`src/api/`, `src/services/`, `src/repositories/`, `src/schemas/`).
   - **Functions & Components:**
     - `schemas/url.py`: `ShortenRequest` (`HttpUrl`, optional `ttl_seconds` constrained `60 <= ttl <= 31536000`), `ShortenResponse` (DTO hiding internal storage fields).
     - `api/dependencies.py`: Wires singleton `URLRepository` and `SystemClock` into `ShortenerService` via FastAPI `Depends` (Dependency Inversion).
     - `api/routes/urls.py`: `POST /v1/urls` (returns `201`), `GET /v1/urls/{code}` (returns `302` `Location` header), `GET /v1/urls` (cursor/keyset pagination: `cursor` + `limit` returning `items` + `next_cursor`—no `OFFSET`!).
   - **Verify:** Inspect auto-generated OpenAPI schema at `/openapi.json`; test invalid URL rejection (`422`) and cursor pagination consistency.
3. **Daily Split:**
   - **DSA (1 hr):** LC 206 *Reverse Linked List* & LC 21 *Merge Two Sorted Lists* & LC 141 *Linked List Cycle*.
   - **HLD (30 min):** Why does `OFFSET 100000 LIMIT 20` degrade linearly and return duplicate/skipped rows under live inserts, while keyset pagination (`WHERE (created_at, id) < (:ts, :id)`) stays $O(\log N)$?
   - **LLD (30 min):** Design a `RadixTrieRouter` (`add_route(method, path_pattern, handler)`, `match(method, path)`) supporting `/v1/users/{id}/orders`.

---

### Day 13 — HTTP Semantics, Idempotency, `ETag` Concurrency & Error Contracts (`4.2`, `4.3`, `4.5`)
1. **Pre-Requisite Learning Resource:**
   - RFC 9110 (*HTTP Semantics*: §9.2.1 Safe Methods, §9.2.2 Idempotent Methods, §13.1 Conditional Requests `If-Match`/`If-None-Match`) + RFC 9457 (*Problem Details for HTTP APIs* §3).
2. **Hands-On Activity:**
   - **What to build:** Add RFC 9457 Problem Details global exception handling, `ETag` conditional caching, and `If-Match` optimistic locking to your FastAPI service.
   - **Functions & Components:**
     - `ProblemDetail` schema: `type`, `title`, `status`, `detail`, `instance`, `request_id`.
     - `register_exception_handlers(app)`: Maps `CodeNotFoundError -> 404`, `CodeExpiredError -> 410 Gone`, `VersionConflictError -> 412 Precondition Failed`, unhandled `Exception -> 500` (never leaking stack traces!).
     - `compute_etag(entity) -> str`: Weak/strong SHA-256 hash of version/updated_at.
     - `GET /v1/urls/{code}/stats`: Checks `If-None-Match` -> returns `304 Not Modified` with empty body on match.
     - `PATCH /v1/urls/{code}`: Requires `If-Match` header -> returns `428 Precondition Required` if missing, `412` on version mismatch, updates atomically on match.
   - **Verify:** Test conditional `GET` (`304`) and lost-update prevention (`412`) using `curl -i -H "If-Match: ..."`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 19 *Remove Nth Node From End of List* & LC 143 *Reorder List* (fast/slow pointers + in-place merge).
   - **HLD (30 min):** Design a long-running async API workflow (`POST /exports -> 202 Accepted + Location: /jobs/{id}`, status polling, `Retry-After`, callback webhook).
   - **LLD (30 min):** Design `ExceptionMapper` + `ErrorBoundaryMiddleware` ensuring domain exceptions map cleanly to HTTP statuses without business layers importing FastAPI.

---

### Day 14 — Middleware Pipeline, Request Context, Config & Graceful Shutdown (`4.1`, `4.5`)
1. **Pre-Requisite Learning Resource:**
   - 12-Factor App (`12factor.net` — Sections I, III, VI, IX Disposability, XI Logs) + Python Docs (*`contextvars` — Context Variables*) + ASGI Lifespan Protocol Spec (`asgi.readthedocs.io/en/latest/specs/lifespan.html`).
2. **Hands-On Activity:**
   - **What to build:** Production middleware pipeline, typed `pydantic-settings` config, structured JSON logging, and graceful shutdown lifecycle in FastAPI.
   - **Functions & Components:**
     - `Settings(BaseSettings)`: Validates `APP_ENV`, `HOST`, `PORT`, `LOG_LEVEL`, `SHUTDOWN_TIMEOUT_SEC` on startup—fails fast if invalid.
     - `RequestContextMiddleware`: Generates/propagates `X-Request-ID` & `trace_id` into `contextvars`, measures duration, logs one structured JSON access event per request (status, method, path, latency_ms, request_id).
     - `ShutdownState`: Tracks `is_draining: bool` and `in_flight_count: int`.
     - `GET /health/live` (returns `200` while process is up) vs `GET /health/ready` (returns `503` immediately when `is_draining=True`).
     - ASGI `lifespan` context manager: On shutdown, flips `is_draining=True`, waits up to `SHUTDOWN_TIMEOUT_SEC` for `in_flight_count == 0`, closes resources cleanly.
   - **Verify & Break:** Hit a `/slow` 4s endpoint, send `SIGTERM` after 1s -> `/health/ready` immediately returns `503` while the in-flight request completes `200 OK`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 23 *Merge k Sorted Lists* (min-heap vs divide-and-conquer—implement the `heapq` solution cleanly).
   - **HLD (30 min):** Sketch a rolling deployment behind an L7 Load Balancer: how readiness probes, connection draining, and `preStop` sleep prevent `502` errors during deploys.
   - **LLD (30 min):** Design an onion-style `MiddlewareChain` (`next_call`) supporting pre-processing, post-processing, and short-circuiting.

---

### Day 15 — HTTP Cookies, Sessions, CORS & Stateful vs Stateless Auth Intro (`2.4`)
1. **Pre-Requisite Learning Resource:**
   - MDN Web Docs (*HTTP Cookies* + *Cross-Origin Resource Sharing (CORS)*) + OWASP Session Management Cheat Sheet (sections on Cookie Attributes: `HttpOnly`, `Secure`, `SameSite`, Session Fixation).
2. **Hands-On Activity:**
   - **What to build:** Add server-side cookie sessions + strict CORS policy to your FastAPI app and test cross-origin behavior from a second local origin.
   - **Functions & Components:**
     - `SessionStore(Protocol)` & `InMemorySessionStore`: Stores `session_id -> SessionData(user_id, created_at, expires_at)`.
     - `POST /auth/login` & `POST /auth/logout`: Generates 256-bit `secrets.token_urlsafe(32)` session ID, **rotates** session ID on login (preventing session fixation), sets cookie with `HttpOnly=True`, `SameSite="lax"`, `Secure` flag driven by env.
     - `configure_cors(app, allowed_origins: list[str])`: Explicitly whitelists origins (never `*` with `allow_credentials=True`).
   - **Verify:** Serve a dummy `index.html` on port `3000` calling port `8000` via `fetch()`—observe browser CORS preflight `OPTIONS` failure first, then fix and inspect `Set-Cookie` / `Access-Control-*` headers.
3. **Daily Split:**
   - **DSA (1 hr):** LC 704 *Binary Search* & LC 33 *Search in Rotated Sorted Array* — invariant `left <= right` vs `left < right`.
   - **HLD (30 min):** Compare Sticky Sessions (L7 LB affinity) vs Shared External Session Store (Redis) vs Stateless Signed Tokens (JWT)—focus on horizontal scaling, revocation, and failure impact.
   - **LLD (30 min):** Design `SessionManager` (`create`, `validate`, `rotate`, `revoke`, `purge_expired`) with sliding vs absolute expiration.

---

### Day 16 — Real-Time Protocols: SSE, WebSockets & gRPC Concepts (`2.6`)
1. **Pre-Requisite Learning Resource:**
   - *HPBN* Chapter 16 (*Server-Sent Events*) & Chapter 17 (*WebSockets*: Upgrade handshake, framing, head-of-line & scaling) + gRPC Official Docs (*Core Concepts, Architecture and Lifecycle*: unary, streaming, deadlines).
2. **Hands-On Activity:**
   - **What to build:** Add an SSE live-click stream and a WebSocket echo/heartbeat endpoint to your FastAPI app; compare framing and backpressure.
   - **Functions & Components:**
     - `EventBroadcaster`: Maintains per-subscriber bounded `asyncio.Queue(maxsize=50)` (prevents slow consumers from OOMing the server!).
     - `GET /v1/urls/{code}/stream` (SSE): Streams `text/event-stream` `data: {...}\n\n` events whenever a short link is clicked, handling client disconnect cleanly.
     - `WS /v1/ws/echo`: Implements WebSocket `accept()`, periodic ping/heartbeat timeout, and bounded message size check.
   - **Verify:** Connect 2 terminals (`curl -N` for SSE) while clicking links; pause one reader to test bounded-queue slow-consumer drop/backpressure.
3. **Daily Split:**
   - **DSA (1 hr):** LC 875 *Koko Eating Bananas* & LC 1011 *Capacity To Ship Packages Within D Days* — Binary Search on Answer pattern.
   - **HLD (30 min):** Design a live Rider Location Push service for 100k concurrent mobile apps: Long Polling vs SSE vs WebSockets (stateful connection termination + Redis Pub/Sub fan-out).
   - **LLD (30 min):** Design a `TopicBroadcaster` with per-client bounded queues (`DROP_OLDEST` vs `DISCONNECT_SLOW_CLIENT` policy).

---

### Day 17 — Provision Azure Free Tier VM, Linux Hardening & `systemd` (`1.1`, `1.5`, `15.3`)
1. **Pre-Requisite Learning Resource:**
   - Azure Docs (*Linux VM Quickstart & Network Security Groups*) + `man 5 systemd.service` & `man 5 systemd.unit` + Ubuntu Server Guide (*UFW Firewall*).
2. **Hands-On Activity:**
   - **What to build:** Provision your **Azure Free Tier `B1s` Ubuntu VM**, set a **$5 Azure Cost Budget Alert**, lock down NSG + SSH, and run your FastAPI service under `systemd`.
   - **Functions & Components:**
     - Azure NSG Rules: Allow TCP `22` *only* from your home public IP; allow TCP `80` and `443` from `Any`; deny all other inbound.
     - Non-root service user (`svc_api`) owning `/opt/urlshortener` with `uv` virtual environment.
     - `/etc/systemd/system/urlshortener.service`: Contains `[Unit]` (`After=network.target`), `[Service]` (`User=svc_api`, `WorkingDirectory=...`, `EnvironmentFile=/etc/urlshortener.env`, `ExecStart=... uvicorn ... --host 127.0.0.1 --port 8000`, `Restart=always`, `RestartSec=3`, `TimeoutStopSec=15`, `KillSignal=SIGTERM`), `[Install]`.
   - **Verify & Break:** Reboot the Azure VM and confirm the API auto-starts; run `systemctl status`, `journalctl -u urlshortener -n 50`, and `ss -tulpn` (verify port `8000` listens *only* on `127.0.0.1`, unreachable externally).
3. **Daily Split:**
   - **DSA (1 hr):** LC 153 *Find Minimum in Rotated Sorted Array* & LC 4 *Median of Two Sorted Arrays* (analyze partition binary search).
   - **HLD (30 min):** Why should an application process bind to `127.0.0.1` behind a reverse proxy rather than `0.0.0.0:8000` on a public VM?
   - **LLD (30 min):** Design a `HealthProbeAggregator` checking multiple downstream checks (disk writable, memory threshold, DB ping) with per-check timeouts.

---

### Day 18 — DNS Routing, Nginx Reverse Proxy & L7 Request Buffering (`2.1`, `2.5`)
1. **Pre-Requisite Learning Resource:**
   - NGINX Official Docs (*Admin Guide: Reverse Proxy*, `proxy_pass`, `proxy_set_header`, `proxy_read_timeout`, upstream buffering) + Cloudflare Learning Center (*What is a Reverse Proxy? L4 vs L7 Load Balancing*).
2. **Hands-On Activity:**
   - **What to build:** Point a domain/subdomain `A` record (e.g., DuckDNS/owned domain) to your Azure B1s public IP and place **Nginx** in front of `127.0.0.1:8000`.
   - **Functions & Components:**
     - DNS setup + verification via `dig +trace <yourdomain>` and `dig A <yourdomain> @1.1.1.1`.
     - `/etc/nginx/sites-available/urlshortener`: Configures `server_name`, `location /`, `proxy_pass http://127.0.0.1:8000`, forwards `Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`, sets `client_max_body_size 1m`, `proxy_connect_timeout 3s`, `proxy_read_timeout 10s`.
     - Custom JSON Nginx access log format capturing `$status`, `$request_time`, `$upstream_response_time`.
   - **Verify & Break:** Stop `urlshortener.service` and hit `http://<yourdomain>/health` -> observe Nginx `502 Bad Gateway` in `/var/log/nginx/error.log`. Restart it, hit a 12s endpoint -> observe `504 Gateway Timeout` and distinguish `request_time` vs `upstream_response_time`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 981 *Time Based Key-Value Store* (HashMap + Binary Search on timestamps) & LC 34 *Find First and Last Position*.
   - **HLD (30 min):** Contrast L4 Load Balancing (TCP passthrough / Direct Server Return) vs L7 Load Balancing (TLS termination, path/header routing, HTTP retries).
   - **LLD (30 min):** Design a `RoundRobinLoadBalancer` and `WeightedLeastConnectionsBalancer` with active health-check marking (`mark_unhealthy`, `mark_healthy`).

---

### Day 19 — TLS 1.3 Internals, X.509 Certificates & Let's Encrypt (`2.5`)
1. **Pre-Requisite Learning Resource:**
   - *HPBN* Chapter 4 (*Transport Layer Security (TLS)*: handshake, forward secrecy, certificates, ALPN, OCSP stapling) + RFC 8446 §1–2 (TLS 1.3 1-RTT handshake & 0-RTT replay risk).
2. **Hands-On Activity:**
   - **What to build:** Terminate HTTPS/TLS 1.3 at Nginx on your Azure VM using Let's Encrypt (`certbot`) and inspect every cryptographic artifact.
   - **Functions & Components:**
     - Run `certbot --nginx -d <yourdomain>` to complete the ACME HTTP-01 challenge.
     - Inspect generated Nginx SSL directives: `ssl_protocols TLSv1.2 TLSv1.3`, HTTP->HTTPS `301` redirect, `Strict-Transport-Security` (HSTS), OCSP stapling.
     - Inspect cert renewal timer (`systemctl list-timers | grep certbot`).
   - **Verify:** From your main laptop, run `openssl s_client -connect <yourdomain>:443 -servername <yourdomain> -tls1_3 -showcerts` to inspect the X.509 Subject Alternative Name (SAN), issuer CA chain, ECDHE key exchange, and negotiated cipher; run `curl -v https://<yourdomain>/health`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 215 *Kth Largest Element in an Array* (min-heap of size $K$ + Quickselect analysis) & LC 703 *Kth Largest Element in a Stream*.
   - **HLD (30 min):** Trace why TLS uses asymmetric crypto (ECDHE + X.509 signatures) *only* during the handshake and symmetric AES-GCM/ChaCha20 afterward, plus why 0-RTT resumption is unsafe for non-idempotent `POST` requests.
   - **LLD (30 min):** Design an internal `CertificateRotationWatcher` that reloads TLS contexts safely when rotated cert files appear on disk.

---

### Day 20 — Mutual TLS (mTLS), TCP Congestion Control & QUIC Concepts (`2.2`, `2.5`)
1. **Pre-Requisite Learning Resource:**
   - Cloudflare Blog — *What is mTLS?* & *BBR Congestion Control* + RFC 9000 §1–3 (QUIC streams & connection migration).
2. **Hands-On Activity:**
   - **What to build:** Create a tiny local private CA on your Ubuntu 16 GB VM using `openssl` and configure **mTLS** between `curl` (client cert) and a local Nginx server.
   - **Functions & Components:**
     - Script/runbook generating: Root CA key+cert -> Server CSR+cert signed by CA -> Client CSR+cert signed by CA.
     - Local Nginx config on Ubuntu VM: `ssl_client_certificate ca.crt` and `ssl_verify_client on`.
   - **Verify & Break:** Call with `curl` *without* `--cert/--key` -> verify TLS alert rejection (`400 No required SSL certificate was sent`); call *with* `--cert client.crt --key client.key` -> `200 OK`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 973 *K Closest Points to Origin* & LC 1046 *Last Stone Weight* (Heap mechanics).
   - **HLD (30 min):** How do service-to-service microservices verify caller identity without passwords/API keys (mTLS + short-lived SPIFFE/X.509 certs)?
   - **LLD (30 min):** Design a `CongestionWindowSimulator` implementing TCP Slow Start (exponential up to `ssthresh`), Congestion Avoidance (+1 MSS/RTT), and AIMD multiplicative decrease on packet loss.

---

### Day 21 — Month 1 Midterm Consolidation: Live Traffic & Socket Inspection (`1.1`, `1.5`, `2.1–2.5`)
1. **Pre-Requisite Learning Resource:**
   - Brendan Gregg — *Linux Performance Analysis in 60,000 Milliseconds* (`brendangregg.com/Articles/Netflix_Linux_Perf_60s.html`) + `man 8 ss`.
2. **Hands-On Activity:**
   - **What to build:** Load-test your live Azure `https://<yourdomain>` API from your main laptop while inspecting live kernel/process telemetry over SSH.
   - **Functions & Components:**
     - Python load-generator script (`src/tools/burst_client.py`): Uses `httpx` to send 500 `POST /v1/urls` + 2,000 `GET` requests and computes `min`, `p50`, `p95`, `p99`, and error rates.
     - Telemetry capture on Azure VM during load: `vmstat 1`, `ss -s`, `top -H -p <uvicorn_pid>`, and Nginx `$upstream_response_time` percentiles parsed from logs.
   - **Verify:** Compare latency of HTTP/1.1 connection-per-request vs connection reuse (`httpx.Client`) against TLS—quantify the exact TLS handshake overhead saved!
3. **Daily Split:**
   - **DSA (1 hr):** Timed Re-Solve (no notes): LC 146 *LRU Cache* + LC 76 *Minimum Window Substring*.
   - **HLD (30 min):** Explain every metric from `vmstat 1` (`r`, `b`, `si/so`, `us`, `sy`, `wa`, `id`) and `ss -s` during an API slowdown.
   - **LLD (30 min):** Design a streaming `LatencyHistogram` (fixed buckets + exact reservoir/percentile summary) for per-endpoint `p50`/`p95`/`p99`.

---

### Day 22 — Linux Namespaces & Control Groups (cgroups v2) From Scratch (`1.5`, `15.1`)
1. **Pre-Requisite Learning Resource:**
   - Julia Evans (`jvns.ca`) / Linux `man 7 namespaces` + `man 7 cgroups` + *Containers From Scratch* (Liz Rice conceptual walkthrough of `unshare` + `chroot` + cgroups).
2. **Hands-On Activity:**
   - **What to build:** Manually create an isolated "container" process on your Ubuntu 16 GB VM **without Docker** using Linux primitives.
   - **Functions & Components:**
     - Step 1: Run `sudo unshare --pid --fork --mount-proc --uts bash` -> change `hostname isolated-box` and run `ps aux` (notice PID 1!).
     - Step 2: Inspect `/sys/fs/cgroup/` (`cgroup.controllers`, `memory.max`, `cpu.max`) and create a manual cgroup slice capping a Python process to `50 MB` RAM and `20%` CPU (`20000 100000`).
     - Step 3: Inspect `lsns` from the host shell to see the isolated PID/UTS/NET namespaces alongside host processes.
   - **Break Intentionally:** Inside the memory-capped cgroup, run a Python script allocating 100 MB -> watch kernel OOM killer terminate it cleanly without hurting the host VM.
3. **Daily Split:**
   - **DSA (1 hr):** LC 295 *Find Median from Data Stream* (Two Heaps: max-heap left, min-heap right).
   - **HLD (30 min):** How does Kubernetes enforce CPU `limits` (CFS quota throttling -> latency spikes!) vs Memory `limits` (hard OOMKill)?
   - **LLD (30 min):** Design a `ResourceQuotaEnforcer` tracking per-tenant CPU/memory reservations with admission `can_schedule(pod_req)` checks.

---

### Day 23 — OverlayFS, Image Layers & Multi-Stage Dockerfiles (`1.5`, `15.1`)
1. **Pre-Requisite Learning Resource:**
   - Docker Docs (*OverlayFS storage driver* + *Multi-stage builds* + *Docker security best practices*) + OCI Image Spec overview (`github.com/opencontainers/image-spec/blob/main/layer.md`).
2. **Hands-On Activity:**
   - **What to build:** Experiment with raw `mount -t overlay` on Ubuntu VM, then write a minimal hardened multi-stage `Dockerfile` for your FastAPI service.
   - **Functions & Components:**
     - Raw OverlayFS check: Create `lower/`, `upper/`, `work/`, `merged/` directories, mount overlay, modify a file from `lower/` inside `merged/`, and inspect `upper/` (Copy-on-Write!) and whiteout files on delete.
     - `Dockerfile`: Stage 1 (`builder`) installs dependencies via `uv` into `.venv`; Stage 2 (`runtime`) copies `.venv` + `src/` onto `python:3.12-slim`, creates non-root UID `10001`, sets `USER 10001`, `HEALTHCHECK`, and `exec uvicorn` (so PID 1 receives `SIGTERM`!).
   - **Break Intentionally:** Run `docker run` with `--read-only --tmpfs /tmp` and verify your app doesn't fail trying to write logs/caches to the root filesystem. Compare layer cache busting when copying `.` before `pyproject.toml`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 355 *Design Twitter* (HashMap + Heap $K$-way merge of user tweet timelines).
   - **HLD (30 min):** Why does putting `exec` (JSON array `CMD ["uvicorn", ...]`) matter for PID 1 signal handling inside Docker, and how do layered CoW filesystems prevent 100 containers from duplicating base OS disk space?
   - **LLD (30 min):** Design a `ContentAddressableLayerStore` (`put_layer(tar_bytes) -> sha256`, `mount_stack(layer_hashes)`) modeling immutable layer deduplication.

---

### Day 24 — Container Networking: `veth`, Linux Bridges & Docker Compose (`15.1`)
1. **Pre-Requisite Learning Resource:**
   - Docker Docs (*Bridge network driver* & *Networking overview*) + Linux `man 4 veth` & `man 8 ip-link`.
2. **Hands-On Activity:**
   - **What to build:** Inspect how two containers actually talk over virtual ethernet (`veth`) pairs and a Linux bridge on your Ubuntu VM, and package your API + Nginx in `docker-compose.yml`.
   - **Functions & Components:**
     - Run two containers on a custom Docker bridge network; inspect `ip link`, `bridge link`, `iptables -t nat -L` (SNAT/DNAT!), and `docker network inspect`.
     - `docker-compose.yml`: Defines `api` (bound only to internal Docker network, `mem_limit: 256m`) and `nginx` (exposed on `80`), sharing a Docker DNS name `http://api:8000`.
   - **Verify & Break:** Exec into `nginx` and run `getent hosts api` (Docker embedded DNS `127.0.0.11`). Break the `depends_on` healthcheck and verify Nginx fails cleanly until `api` is healthy.
3. **Daily Split:**
   - **DSA (1 hr):** LC 621 *Task Scheduler* (greedy/max-heap cooldown scheduling) & LC 767 *Reorganize String*.
   - **HLD (30 min):** Trace a packet leaving Container A -> `veth0` -> `docker0` bridge -> `iptables` NAT -> host NIC -> Azure VM.
   - **LLD (30 min):** Design an `InMemoryServiceRegistry` (`register(name, ip, port, ttl)`, `heartbeat()`, `resolve(name)`) for dynamic service discovery.

---

### Day 25 — Interval & Scheduling Algorithms + Delayed Job Primitives (`13.2`)
1. **Pre-Requisite Learning Resource:**
   - *Data Structures*: Python `heapq` documentation (*Priority Queue Implementation Notes*) + George Varghese & Tony Lauck — *Hashed and Hierarchical Timing Wheels* (§1–2 overview).
2. **Hands-On Activity:**
   - **What to build:** A pure-Python `DelayedTaskScheduler` (`src/scheduler/delayed.py`) backed by a min-heap + `Clock` injection.
   - **Functions & Components:**
     - `ScheduledTask` dataclass (`orderable` on `run_at_ts`, `seq_id`, `task_id`, `payload`, `cancelled: bool`).
     - `DelayedTaskScheduler`:
       - `schedule(task_id: str, delay_sec: float, payload: dict) -> None`: Pushes to min-heap.
       - `cancel(task_id: str) -> bool`: Lazy deletion (marks `cancelled=True` in map so $O(\log N)$ stays intact without $O(N)$ heap scan!).
       - `pop_due_tasks() -> list[ScheduledTask]`: Pops while `heap[0].run_at_ts <= clock.now()`, skipping cancelled entries.
   - **Verify:** Benchmark 50k scheduled tasks with random delays and 10k cancellations; verify exact chronological execution order.
3. **Daily Split:**
   - **DSA (1 hr):** LC 56 *Merge Intervals* & LC 253 *Meeting Rooms II* (sort + min-heap end-times).
   - **HLD (30 min):** Compare a Min-Heap / Redis `ZSET` poller vs a Hierarchical Timing Wheel for scheduling 10M delayed webhook retries.
   - **LLD (30 min):** Add retry-with-backoff and max-concurrent-execution semaphore to `DelayedTaskScheduler`.

---

### Day 26 — GitHub Actions CI/CD & Immutable Artifact Promotion (`15.3`)
1. **Pre-Requisite Learning Resource:**
   - GitHub Actions Docs (*Building and testing Python*, *Publishing Docker images to GHCR*, *Deployment environments*).
2. **Hands-On Activity:**
   - **What to build:** Build a real CI/CD pipeline (`.github/workflows/deploy.yml`) that turns `git push main` into a tested, immutable SHA-tagged Docker deploy onto your Azure B1s VM.
   - **Functions & Components:**
     - Job 1 (`ci`): Runs `uv sync --frozen`, `ruff check`, `mypy src`, `pytest`.
     - Job 2 (`build-push`): Builds multi-stage Docker image tagged with `${{ github.sha }}` (never `:latest` alone!) and pushes to GHCR (`ghcr.io`).
     - Job 3 (`deploy`): SSHs into Azure VM using GitHub Encrypted Secret SSH key, pulls `ghcr.io/...:${SHA}`, swaps container atomically, hits `https://<yourdomain>/health/ready`, and rolls back to previous SHA if health check fails!
   - **Break Intentionally:** Introduce a deliberate `mypy` error -> verify the pipeline blocks deployment before building an image.
3. **Daily Split:**
   - **DSA (1 hr):** LC 57 *Insert Interval* & LC 435 *Non-overlapping Intervals*.
   - **HLD (30 min):** Compare In-Place SSH Deploy vs Blue-Green Deployment vs Canary Progressive Delivery (traffic shifting + automated rollback on error-rate spike).
   - **LLD (30 min):** Design a `DeploymentRollbackManager` state machine tracking active SHA, previous SHA, health probe count, and rollback trigger.

---

### Day 27 — OpenAPI Contract-First Validation & Breaking-Change Detection (`4.3`, `16.1`)
1. **Pre-Requisite Learning Resource:**
   - OpenAPI 3.1 Specification Overview (`spec.openapis.org/oas/latest.html` §1–4) + Google Cloud API Design Guide (*Versioning & Backwards Compatibility*).
2. **Hands-On Activity:**
   - **What to build:** Export your FastAPI `openapi.json` snapshot into git and write a contract-regression test (`tests/test_api_contract.py`) that fails on breaking API changes.
   - **Functions & Components:**
     - `export_openapi_schema()` script writing deterministic sorted JSON.
     - `detect_breaking_changes(old_spec: dict, new_spec: dict) -> list[str]`: Checks if any existing path/method was removed, any new *required* request field was added, or any existing response property changed type/was removed.
   - **Verify:** Test removing a response field -> regression test catches the breaking change; test adding an optional field -> passes cleanly.
3. **Daily Split:**
   - **DSA (1 hr):** LC 226 *Invert Binary Tree*, LC 104 *Maximum Depth*, LC 102 *Binary Tree Level Order Traversal* (BFS queue).
   - **HLD (30 min):** Compare URI versioning (`/v1/`, `/v2/`) vs Header/Media-Type versioning (`Accept: application/vnd.api.v2+json`) and API deprecation lifecycles (`Sunset` header).
   - **LLD (30 min):** Design an `APIVersionRouter` supporting backwards-compatible response adapters (`v2` internal model -> `v1` DTO).

---

### Day 28 — gRPC & Protocol Buffers vs REST/JSON (`2.6`, `4.2`)
1. **Pre-Requisite Learning Resource:**
   - Protocol Buffers Docs (*Encoding: Varints & Wire Types* `protobuf.dev/programming-guides/encoding/`) + gRPC Docs (*Deadlines & Cancellation*).
2. **Hands-On Activity:**
   - **What to build:** Define a `.proto` contract (`url_shortener.proto`), compile Python stubs with `grpcio-tools`, and run a gRPC server + client on your Ubuntu VM with deadlines and metadata propagation.
   - **Functions & Components:**
     - `url_shortener.proto`: Defines `ShortenRequest`, `ShortenResponse`, and `rpc Shorten` + `rpc StreamClicks (stream)` using explicit field numbers (`= 1`, `= 2`) and `reserved` fields for evolution.
     - `grpc_server.py`: Implements `ShortenerServicer` checking `context.is_active()` and `context.time_remaining()` before doing simulated DB work.
     - `grpc_client.py`: Calls with `timeout=0.5` and inspects binary payload byte size vs equivalent JSON string.
   - **Break Intentionally:** Make server sleep `1.0s` while client sets `timeout=0.3s` -> observe `DEADLINE_EXCEEDED` and verify server aborts remaining work via `context.is_active()`.
3. **Daily Split:**
   - **DSA (1 hr):** LC 98 *Validate Binary Search Tree* & LC 235 *Lowest Common Ancestor of a BST*.
   - **HLD (30 min):** Why do internal microservices (e.g., Uber backend mesh) prefer gRPC/Protobuf over HTTP/1.1+JSON, while public APIs stay REST/JSON?
   - **LLD (30 min):** Design a `DeadlineContext` propagated across 3 downstream service hops (Service A -> B -> C), subtracting elapsed time at each hop.

---

### Day 29 — Month 1 Live Production Failure Drills (`1.1`, `1.5`, `2.3–2.5`, `4.5`)
1. **Pre-Requisite Learning Resource:**
   - Review your Month 1 `notes.md` + Nginx Troubleshooting Docs (`nginx.org/en/docs/debugging_log.html`).
2. **Hands-On Activity:**
   - **What to build:** Execute 4 live break-and-diagnose drills across your Ubuntu VM + Azure B1s production stack without looking at solutions:
     1. Exhaust file descriptors (`ulimit -n 32`) on a local server process and observe `EMFILE: Too many open files` (`1.3`, `1.5`).
     2. Misconfigure Nginx `upstream` port -> diagnose `502` from logs + `ss -tulpn`.
     3. Break DNS resolution locally via `/etc/hosts` or wrong `Host` header -> diagnose via `curl -v --resolve`.
     4. Send `SIGTERM` under active `httpx` traffic -> verify zero `5xx` dropped requests via graceful draining.
3. **Daily Split:**
   - **DSA (1 hr):** LC 208 *Implement Trie (Prefix Tree)* & LC 211 *Design Add and Search Words Data Structure*.
   - **HLD (30 min):** Draw your exact Month 1 production architecture (Developer -> GitHub Actions -> GHCR -> Azure VM -> NSG -> Nginx TLS 1.3 -> Docker FastAPI -> Logs) and mark every failure point.
   - **LLD (30 min):** Implement `RadixTrieRouter` from Day 12's design in pure Python supporting static + `:param` segments.

---

### Day 30 — Month 1 Capstone Audit & Timed Assessment
1. **Pre-Requisite Learning Resource:**
   - Re-read [Curriculum.md](./Curriculum.md) Sections `1.1`, `1.5`, `2.1–2.6`, `3.4–3.5`, `4.1–4.3`, `4.5`, `15.1`, `15.3`.
2. **Hands-On Activity:**
   - **What to build:** Clean-room drill on Ubuntu VM: create a brand-new minimal HTTP service, containerize it (non-root, multi-stage), front it with Nginx + local TLS cert, and confirm zero errors—all strictly from memory (no copying old configs, no AI).
   - **Verify:** Audit `htop`, `df -h`, `docker ps`, and Azure billing dashboard (`$0.00` spent).
3. **Daily Split:**
   - **DSA (1 hr):** Timed 60-min Mock (zero hints): LC 146 *LRU Cache* + LC 295 *Find Median from Data Stream*.
   - **HLD (30 min):** Timed 30-min Blank-Page Design: **Pastebin / Ephemeral Text & File Sharing Service** (API, storage split, expiry cleanup, caching, CDN/reverse proxy).
   - **LLD (30 min):** Timed 30-min Blank-Page Design: **In-Memory Rate Limiter supporting multiple algorithms (`TokenBucket`, `FixedWindow`) behind one `RateLimiter(Protocol)`**.
