# 📊 CAPACITY & SCALE NUMBERS
> Know these cold. Use them in back-of-envelope estimations.
> Every limit comes from: CPU, Memory, Disk I/O, or Network.

---

## The Mental Model

```
Every service limit comes from one of four physical constraints:
  CPU      — processing power, thread count
  Memory   — RAM available  
  Disk I/O — read/write throughput
  Network  — bandwidth in/out

When a service hits its limit, one of these is the bottleneck.
Knowing which one tells you how to scale it.
```

---

## Back-of-Envelope Formulas

```
QPS (average) = (DAU × req/user/day) / 86,400
QPS (peak)    = QPS average × 3

Storage/day   = QPS × 86,400 × message_size

Memory needed = users × data_per_user

10M DAU × 10 req/day  = ~1,200 QPS avg = ~3,500 QPS peak
100M DAU × 10 req/day = ~12,000 QPS avg = ~35,000 QPS peak
```

---

## Application Servers / Pods

### Standard Pod (2 vCPU, 4GB RAM)
```
Threaded (Django/Spring):   500-2,000 RPS
Async (Node/FastAPI):       5,000-10,000 RPS
Concurrent threads:         50-200 practical
Memory per thread:          ~1-2MB
```

### Why async is faster
No context switching. Event loop handles thousands of connections with one thread — as long as operations are non-blocking.

---

## Databases

### PostgreSQL (single primary)
```
Reads:        5,000-10,000 QPS
Writes:       1,000-3,000 QPS
Connections:  100-500 (use PgBouncer beyond this)
Storage:      TBs easily
Scale read:   Add replicas (each adds 5-10K read QPS)
Scale write:  Shard (each shard adds 1-3K write QPS)
```

### Redis (single node)
```
Reads/Writes: 100,000-1,000,000 ops/sec
Connections:  10,000+ concurrent clients
Limit:        RAM size
Latency:      ~1ms
```

### Cassandra (per node, 8 vCPU 32GB)
```
Writes:    10,000-30,000/sec  ← writes faster than reads (LSM tree)
Reads:     5,000-15,000/sec
Storage:   1-4TB per node
Scaling:   LINEAR — double nodes = double throughput
```

### DynamoDB
```
Latency:   < 10ms guaranteed
Scale:     Essentially unlimited (AWS manages it)
Use:       When you need guaranteed latency at any scale
```

### MongoDB (single node)
```
Reads:     10,000-50,000 QPS
Writes:    5,000-20,000 QPS
```

---

## Message Queues

### Kafka (single broker, 8 vCPU 32GB)
```
Throughput:  100MB/sec - 1GB/sec
Messages:    100,000-1,000,000/sec
Latency:     2-10ms
Retention:   Days to weeks (storage-bound)
Consumers:   1 per partition per consumer group
```

### Kafka (3-broker cluster, 100 partitions)
```
Messages:    500,000-5,000,000/sec
Consumers:   Up to 100 per consumer group
```

### Azure Event Hub
```
Basic:     20MB/sec ingress
Premium:   Scales to 1,000,000+ events/sec
Retention: 1-90 days
```

### Azure Service Bus
```
Standard:  ~115 ops/sec sustained
Premium:   1,000-4,000 messages/sec per unit
Max size:  256KB (Standard), 100MB (Premium)
```

### Redis Queue (Celery)
```
Enqueue:   50,000-100,000 tasks/sec
```

---

## Object Storage (S3 / Azure Blob)

```
PUT rate:   3,500/sec per prefix
GET rate:   5,500/sec per prefix
Throughput: Multiple GB/sec per prefix
Storage:    Unlimited
File size:  Up to 5TB per object
Latency:    100-200ms cold, 10-50ms warm
With CDN:   5-50ms from nearest edge
```

---

## Load Balancers

### Nginx (single instance)
```
RPS:         50,000-100,000 (static)
             10,000-50,000 (proxying)
Connections: 10,000-100,000 concurrent
```

### AWS ALB
```
RPS:         Millions (auto-scales)
Latency:     < 1ms added
```

---

## WebSockets

```
Single server (8 vCPU, 16GB):
  Connections:   10,000-100,000 concurrent
  Memory/conn:   ~2-10KB
  
K8s pod (2 vCPU, 4GB):
  Connections:   10,000-50,000 concurrent

1M concurrent connections = ~20 dedicated WebSocket servers
```

**Bottleneck:** Memory (each connection holds state), not CPU.

---

## Kubernetes

```
Pods per node:      110 (default limit)
Nodes per cluster:  5,000 (default limit)
Total pods:         550,000 max

Small node  (2 vCPU, 8GB):    10-20 pods
Medium node (8 vCPU, 32GB):   40-80 pods
Large node  (32 vCPU, 128GB): 100+ pods
```

---

## The Scale Ladder

```
Writes/sec     Action
─────────────────────────────────────────
< 1,000        Single Postgres — fine
1,000-5,000    + PgBouncer connection pooling
5,000-20,000   + Read replicas
20,000-100K    Shard Postgres or switch to Cassandra
> 100K         Cassandra / DynamoDB

Messages/sec   Action
─────────────────────────────────────────
< 1,000        Celery + Redis — fine
1,000-10K      RabbitMQ or Service Bus
10K-100K       Kafka (small cluster)
> 100K         Kafka multi-broker or Event Hub Premium

Concurrent     Action
connections
─────────────────────────────────────────
< 10K          Single app server
10K-100K       Multiple pods behind LB
100K-1M        Dedicated WebSocket servers
> 1M           Distributed WS cluster
```

---

## Interview Estimation Template

```
1. State: "10M DAU, ~50 requests/user/day"
2. Calculate: 10M × 50 / 86,400 = ~5,800 QPS avg
3. Peak: 5,800 × 3 = ~17,000 QPS peak
4. Check: Single Postgres handles ~5K writes/sec
5. Decide: "Mostly reads — I'd add 2 read replicas.
            Write volume is ~2K/sec, single primary handles it."
```

---

## Key Numbers to Memorize

```
Redis:           1,000,000 ops/sec (in-memory superpower)
Postgres:        5,000-10,000 QPS (with indexing)
Kafka:           1,000,000 messages/sec (per broker)
Nginx:           50,000-100,000 RPS
WebSocket:       100,000 connections per server
K8s node:        110 pods max
K8s cluster:     5,000 nodes max
Peak multiplier: 3x average QPS
Threads/pod:     50-200 practical
Memory/thread:   ~1-2MB
Memory/WS conn:  ~2-10KB
S3 GET rate:     5,500/sec per prefix
```

---

## Thread Pools

### What a Thread Is
A thread is a unit of execution within a process. Multiple threads share the same memory space. While Thread 1 waits for a DB response (I/O wait, CPU idle), Thread 2 can use the CPU — threads enable concurrency during waits.

### Why Thread Pool (Not New Thread Per Request)
Creating a thread isn't free:
- ~1-2MB stack memory per thread
- OS scheduler overhead for each new thread
- 10,000 concurrent requests = 10-20GB just for thread stacks

Thread pool solution: pre-create N threads at startup, reuse them.
```
Pool of 50 threads:
  Request arrives → grab idle thread → handle → return thread to pool
  All 50 busy → request 51 WAITS in queue until one frees up
```

### Thread Pool = Your Concurrency Ceiling
50 threads = 50 requests in flight max, regardless of incoming traffic.

**This is the thread starvation root cause:**
```
SendGrid down (30s timeout per request):
  Thread 1-50 → all stuck waiting on SendGrid
  Request 51+ → no threads available → queue grows → system appears dead
  
Circuit breaker fixes: converts 30s waits → 1ms fails → threads released immediately
```

### Sizing Formula
```
CPU-bound work:   threads ≈ number of CPU cores
                  (more threads = context switching overhead, no benefit)

I/O-bound work:   threads = cores × (1 + wait_time/compute_time)
  Example: 4 cores, 90% waiting 10% compute:
  threads = 4 × (1 + 9) = 40 threads optimal
```
Web servers are I/O-bound → run 50-200 threads on 2-4 core machines.

### Thread Pool Sizing in Practice
```
Gunicorn (Python):    workers × threads = concurrency
                      e.g., 4 workers × 10 threads = 40 concurrent requests

DB connection pool:   MUCH smaller than thread count
                      200 app threads, 20 DB connections (PgBouncer)
                      DB connections are expensive — pool and share them

Celery workers:       --concurrency=10 → 10 tasks simultaneously
                      11th task waits in Redis queue

Within a request:     ThreadPoolExecutor for parallel sub-calls
                      3 microservice calls in parallel → max(latency) not sum
```

### Threads vs Processes vs Async
```
Threads (Spring, Django):  OS threads, share memory, 500-2K RPS
Processes (Gunicorn, Celery prefork): separate memory, heavier, more isolated
Async (Node, FastAPI):     single thread, non-blocking I/O, 5-10K RPS
                           thousands of concurrent connections, one thread
```
Async avoids thread overhead entirely for I/O-bound — why async is faster.

### Everything Else Is Protecting the Thread Pool
```
Rate limiter   → prevents overwhelming thread pool with too many requests
Circuit breaker → prevents thread pool exhaustion from slow dependencies
Connection pool → manages DB connections shared across threads
HPA (K8s)      → scales pods (each with their own thread pool) based on load
Load balancer  → distributes across multiple pods, each with their own pool
```

---

## CONN_MAX_AGE — Common Misconception Corrected

```python
DATABASES = {'default': {'CONN_MAX_AGE': 600}}
```
**This is 600 SECONDS (10 minutes) — a unit of TIME, NOT a count of connections or requests.**

```
Without it: every request opens a NEW DB connection, closes after
With it:    connection reused for up to 600s before reopening
Per-process: each Gunicorn worker maintains its own reused connection
```
Does NOT mean "max 600 DB interactions." Has nothing to do with concurrent capacity.

**What actually determines concurrent DB capacity:**
```
Gunicorn workers × threads per worker ≈ concurrent requests possible
≈ concurrent DB connections possible (roughly 1 per active request)

Also bounded by Postgres's own max_connections (default ~100)
```

---

## Threads vs Workers vs CPU Cores (Python-Specific — GIL)

```
CPU core  = HARDWARE — real physical limit of simultaneous execution
Worker    = a PROCESS — separate memory, separate GIL, TRUE parallelism
            across cores (Gunicorn's --workers)
Thread    = lives INSIDE a process, SHARES its memory, limited by GIL
```

### The GIL (Global Interpreter Lock)
```
Only ONE thread per PROCESS can execute Python bytecode at any instant.
GIL is RELEASED during I/O waits (DB calls, network, file I/O).
GIL is NEVER released during CPU computation.

→ Threads help for I/O-bound work (overlapping wait time)
→ Threads give ZERO benefit for CPU-bound work (GIL blocks parallelism)
```
This is WHY Gunicorn's primary scaling lever is `--workers` (processes, bypass GIL), not threads. Threads (`--threads`) are a secondary lever, useful only because most web requests are I/O-bound.

---

## CPU-Bound Work — Threading Doesn't Help, Use This Instead

```python
# WRONG for CPU-heavy work — GIL blocks parallelism, threads run sequentially
ThreadPoolExecutor(max_workers=10)

# RIGHT — separate processes, real parallel CPU execution
from concurrent.futures import ProcessPoolExecutor
with ProcessPoolExecutor(max_workers=4) as executor:  # cap at core count
    results = list(executor.map(heavy_computation, items))

# BETTER for web requests — never block a web worker with heavy compute
@app.task
def heavy_computation_task(item_id): ...

def my_view(request):
    task = heavy_computation_task.delay(item.id)
    return JsonResponse({'task_id': task.id}, status=202)  # async, return immediately
```
```
Rule of thumb:
  I/O-bound, fast        → ThreadPoolExecutor inline, fine
  CPU-bound, fast (<1-2s) → ProcessPoolExecutor inline, capped at core count
  CPU-bound, slow         → Celery, ALWAYS — never block a web worker
```

---

## ThreadPoolExecutor Inside a View — Sizing It Correctly

**This is a DIFFERENT sizing problem than Gunicorn's `(2×cores)+1`.** That formula is for CPU-bound process scaling. A ThreadPoolExecutor for I/O-bound work (e.g., 10 parallel DB updates) is bound by the DOWNSTREAM RESOURCE, not CPU.

```
Rule 1: never exceed your actual task count (10 items → max_workers=10, not 50)
Rule 2: the real ceiling is what you're calling (Postgres connections,
        or an external API's rate limit) — not your own code
```

### Multi-Request Worst Case (the hard part)
```
Worst case total DB connections = Gunicorn workers × ThreadPoolExecutor max_workers
  9 workers × 10 threads = 90 possible simultaneous connections

Check against Postgres max_connections (minus reserve for other tools):
  available ≈ 80 → 90 > 80 → will eventually hit "too many connections"

Safe formula:
  max_workers_per_pool ≈ (available_db_connections - reserve) / gunicorn_workers
  (80) / (9) ≈ 8.8 → use max_workers=8, not 10
```

### This Is the Rate Limiter Problem, Different Costume
Bounding concurrent usage of a shared resource across multiple independent processes — same pattern as the Rate Limiter case study (see SYSTEM_DESIGN.md), just applied to DB connections instead of API requests per user.

```python
# Distributed semaphore via Redis — same INCR pattern as the rate limiter
def acquire_db_slot(max_concurrent=80):
    current = r.incr("active_db_threads")
    if current > max_concurrent:
        r.decr("active_db_threads")
        raise Exception("DB connection budget exhausted")
    return True
```
Enforces the real ceiling directly, rather than trusting static worst-case math.

**Real fix for the DB connection ceiling itself: PgBouncer.** Multiplexes many app-level connections onto fewer real Postgres connections — raises effective capacity dramatically (e.g., 1000 app connections → 20 real Postgres connections).

**If parallel work calls an external API instead of your DB:** the ceiling is THEIR rate limit (often global, not per-server) — same Redis-centralized-counter solution applies.

---

## Max Threads — Practical vs Theoretical

```
OS theoretical ceiling (Linux): governed by ulimit -u, often thousands,
  also bounded by available RAM (each thread ~8MB virtual stack default)

Practical USEFUL ceiling (Python, I/O-bound): tens to low hundreds (50-200)
  — beyond this, scheduling overhead dominates, diminishing returns

CPU-bound: more than core count = ZERO benefit (GIL) — use
  ProcessPoolExecutor or Celery instead of pushing thread count higher
```

### What Happens At the Max
```
Bounded pool — ThreadPoolExecutor(max_workers=10):
  11th task submitted → WAITS in internal queue, no crash, graceful

Unbounded — raw threading.Thread() in a loop, no limit:
  Eventually: RuntimeError: can't start new thread
  (OS pthread_create() fails — EAGAIN, hit ulimit or out of memory)
```
Lesson: always use a bounded pool (`ThreadPoolExecutor` with explicit `max_workers`), never raw unbounded thread creation.

---

## Max Processes — Real Numbers

```
Linux pid_max: theoretical ~4,194,304 on modern 64-bit kernels — never the
  real-world constraint

REAL constraint: MEMORY
  max_processes ≈ available_RAM / memory_per_process (~150MB typical Django worker)

Example: 32GB machine, ~150MB/worker → ~200 workers theoretically possible
  from MEMORY perspective — but CPU formula (2×cores)+1 almost always
  caps you LOWER first (e.g., 17 for 8 cores) — CPU is the binding
  constraint in typical sizing, not memory
```

---

## Concrete Worked Example — Typical Corporate Server (8 vCPU, 32GB RAM)

```
I/O-bound app (typical Django — DB calls, API calls):
  workers = (2×8)+1 = 17
  memory check: 17 × 150MB ≈ 2.5GB — fine, not the constraint
  17 workers × 2 threads = 34 concurrent capacity
  avg request 80ms → RPS ≈ 34/0.08 ≈ 425 RPS sustainable

CPU-bound app (heavy computation in request path):
  DON'T use I/O formula — no wait time to hide behind
  workers ≈ cores (or cores-1) = 7-8, NOT 17
  avg request 500ms (real computation) → RPS ≈ 8/0.5 = 16 RPS

Same machine, same cores — 25x throughput difference, purely from
whether work is I/O-bound or CPU-bound.
```

---

## Request Queueing Chain — What Happens Beyond Capacity

```
1. Kernel socket backlog (lowest layer)
   Gunicorn --backlog (default 2048) = max pending TCP connections
   Backlog full + all workers busy → OS REFUSES new connections outright

2. Nginx in front (if present)
   worker_connections setting, buffers more gracefully
   Returns 502/503 instead of raw "connection refused"

3. Gunicorn workers — your configured ceiling (17, or 8 for CPU-bound)
   All busy → requests wait in backlog queue until one frees up

4. Regular saturation → the fix is NOT infinite vertical tuning
   → HORIZONTAL SCALING: more servers behind a load balancer
   → This is what K8s HPA automates — scale pods based on real-time load
```

---

## How It All Connects — Layered System Throughput

The seeming contradiction: "my server handles 500 RPS" vs "current systems handle 100K RPS" vs "Redis handles 1M ops/sec on one thread" — these are NOT comparable numbers. They describe different LAYERS doing fundamentally different amounts of work per operation.

```
Throughput is inversely proportional to "real work done per request."

CDN              → near-zero work (cached bytes) → millions of RPS
Load Balancer    → near-zero work (forward packets) → 100K+ RPS (REAL number, wrong layer)
App Server       → real business logic, DB calls → hundreds-low-thousands RPS
Redis            → microsecond in-memory lookup → 100K-1M ops/sec (REAL, different unit of work)
Database         → heaviest real work (disk I/O, ACID) → 5,000-10,000 QPS
```

### Layered Request Flow (50,000 RPS total example)
```
50,000 RPS incoming
  ↓
CDN absorbs ~80% (static/cacheable) → millions of RPS capacity, never reaches infra
  ↓ ~10,000 RPS reaches infra
Load Balancer (100K+ RPS capacity — near-zero per-request work)
  ↓ distributed across fleet
App Server FLEET: 20 instances × 500 RPS each = 10,000 RPS aggregate
  (horizontal scaling — NOT one impossibly fast box)
  ↓ each instance calls
Redis Cache (100K-1M ops/sec) — absorbs most reads, DB barely touched
  ↓ cache misses only
Database (5,000-10,000 QPS) — sees the LEAST traffic, by design
```

**The "100K RPS per server" claims are real for CDN/LB/Redis (near-zero work per op) — never real for an app server doing actual business logic.** "System handles 100K RPS" usually means the FLEET aggregate, not one instance.

### The Unifying Principle
```
High-throughput systems aren't built by making one layer impossibly fast.
They're built by:
  1. Minimizing real work per request (caching, CDN)
  2. Parallelizing whatever real work remains (horizontal scaling)

Every case study so far has been a version of this:
  URL Shortener  → cache absorbs reads, DB barely touched
  Rate Limiter   → Redis absorbs the hot-path check, never hits DB
  Notification   → fan-out across many workers, not one fast worker
  Pastebin       → CDN/presigned URLs bypass your server entirely
```

### One-liner
> *"A single app server doing real business logic handles hundreds to low-thousands RPS — that's the honest cost of real work. 100K+ RPS numbers describe CDN, load balancer, or cache layers doing near-zero computation, or an aggregate across a horizontally-scaled fleet. High system throughput comes from minimizing real work (caching/CDN) and parallelizing what remains (horizontal scaling) — not from making any single layer impossibly fast."*
