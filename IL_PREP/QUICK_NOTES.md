# ⚡ QUICK NOTES — Pre-Interview Cheat Sheet
> Read this the morning of your interview. Not for learning — for recalling.

---

## The Mental Checklist (First 2 mins of every question)

```
1. Clarify requirements
   - Functional: what does it do?
   - Non-functional: scale, latency, availability?
   - Per user or global? Read-heavy or write-heavy?
   - What happens on failure / limit exceeded?

2. Estimate scale
   - DAU → QPS → storage
   - 10M DAU × 10 req/day = ~1000 QPS
   - Timestamp/ID/counter = 8 bytes
   - Short string = 100 bytes, Document = 1KB

3. High level design — BOXES FIRST
   - Client → LB → Service → Cache → DB
   - Finish the full diagram before zooming into any component

4. Deep dive — find the bottleneck
   - Where does it break at 10x load?
   - How do you fix that specific layer?

5. Trade-offs — always name them
   - "I'm choosing X over Y because Z,
      and the cost is W which is acceptable here because..."

6. Failure thinking — do this unprompted
   - "What happens when this component goes down?"
```

---

## Caching

**What it is:** Faster intermediate store. Trades staleness for speed.

**Where it sits:**
```
User → Browser cache → CDN → App cache (Redis) → DB → DB buffer pool
```

**Eviction:** LRU (default), LFU, TTL

**Invalidation strategies:**

| Strategy | Write goes to | Risk |
|----------|--------------|------|
| Cache-aside | DB, delete cache | Brief staleness |
| Write-through | DB + cache | Slow writes |
| Write-behind | Cache first, DB async | Data loss |

**When aggressive caching is safe:** Immutable data (URL mappings) — long TTL fine.

---

## Database Scaling

- **Read replicas** = same data, multiple DBs → solves read scale
- **Sharding** = split data, multiple DBs → solves write scale
- **Vertical** = bigger machine. **Horizontal** = more machines.
- Replication lag = ~50-100ms, usually acceptable

---

## CAP Theorem

```
C — Consistency      : every read gets the latest write
A — Availability     : every request gets a response
P — Partition Tolerance : system works despite network failures

P is non-negotiable (network partitions always happen)
Real choice: C or A when partition occurs
```

**CP systems** — consistency over availability
- SQL DBs, HBase, Zookeeper
- Use when: banking, payments, anything where wrong data = disaster

**AP systems** — availability over eventual consistency
- Cassandra, DynamoDB, DNS
- Use when: social feeds, likes, carts — stale data briefly is fine

**Interview formula:**
> *"I'd use Cassandra here — this is AP. Eventual consistency is acceptable because
> [stale likes/feed is fine]. Availability matters more than perfect consistency."*

**Consistency models:**
- Strong consistency = every read sees latest write (CP)
- Eventual consistency = reads may be briefly stale, converge over time (AP)

---

## Consistent Hashing

**Problem with normal hashing:** hash(key) % N — adding/removing a server remaps ~100% of keys → cache invalidated → DB gets hammered.

**Consistent hashing solution:**
- Imagine a ring of 0 → 2³²
- Place servers at positions on the ring (hash their names)
- For any key → hash it → walk clockwise → first server = your server
- Add a server → only ~1/N keys remapped (just the new server's neighbour)

**Virtual nodes:** Each physical server gets multiple positions on ring → even load distribution even as servers are added/removed.

**One-liner:**
> *"Normal hashing remaps ~100% of keys on server change.
> Consistent hashing remaps only ~1/N. Virtual nodes ensure even load."*

**Used in:** Redis Cluster, Cassandra, DynamoDB, CDNs.

---

## Leader-Follower + Quorum

**Leader-Follower:**
- One leader accepts all writes
- Followers replicate from leader, serve reads
- Leader = primary, Follower = replica (same concept, formal name)

**When leader dies → Leader Election:**
- Followers vote for a new leader
- Need quorum (majority) to elect → prevents split-brain

**Quorum = majority of nodes:**
```
5 nodes → quorum = 3
Network splits → group of 3 can elect leader
             → group of 2 cannot (no majority)
Two leaders simultaneously impossible ✅
```

**Split-brain:** Two nodes both think they're leader → data diverges → catastrophic.
**Quorum prevents it:** Two groups cannot both hold majority simultaneously.

**Where it's used:**
- Kafka — every partition has a leader broker
- PostgreSQL with Patroni — automatic failover via quorum
- Zookeeper, etcd — built entirely on leader election

**Interview line:**
> *"If the leader dies, a new one is elected via quorum among followers.
> Majority requirement prevents split-brain — two groups can't both hold majority."*

---

## Rate Limiting

**Fixed window:** 1 counter + 1 timestamp per user. Memory efficient. Boundary exploit risk.
**Sliding window:** List of timestamps. Accurate. 50x more memory.
**Distributed:** All servers → one central Redis. Stateless servers.
**Redis failure:** Fail open (most common). Fail closed (security critical).

---

## Message Queues — Kafka

**Topic** = the channel. Producers write, consumers read.
**Partition** = splits topic for parallelism. Ceiling on consumers per group.
**Consumer Group** = independent subscriber. Gets ALL messages.
**Consumer** = worker within group. Gets assigned partitions.

```
Rule: consumers per group ≤ partitions
Multiple groups = pub/sub (each group gets all messages independently)
```

**Kafka vs Celery+Redis:**
- Simple background jobs → Celery + Redis
- High throughput, multiple consumers, retention → Kafka

---

## URL Shortener

```
POST /urls  { long_url } → { short_url }
GET  /{code}             → HTTP 302 redirect
```

Base62(auto-increment ID). 62⁷ = 3.5 trillion. No collisions.
Read flow: Cache HIT (~1ms) | MISS → DB → populate cache (~20ms)

---

## Notification System

**Three flows:**
```
Direct (1→1):     Event → Queue → Worker → Send
Fan-out (1→many): Event → Fan-out Service → N messages → Workers → Send
Aggregation:      N events → Redis counter → Periodic job → 1 batched notification
```

**Fan-out:** 1 event → Fan-out Service pushes N individual messages → N workers process in parallel.
**Batching:** Many likes → Redis counter → "X people liked your post" (Instagram pattern).

---

## Core Distributed Systems Principles

> **Stateless servers + centralized state = scalable distributed system**
> **Quorum (majority) = the safe way to make distributed decisions**
> **CAP: you must choose C or A when partition occurs**
> **Consistent hashing: minimize remapping when topology changes**

---

## Numbers to Know

| Thing | Value |
|-------|-------|
| Redis read latency | ~1ms |
| DB query latency | ~10-100ms |
| Replication lag | ~50-100ms |
| Integer/timestamp | 8 bytes |
| Short string | 100 bytes |
| Document | 1KB |
| 62⁷ combinations | ~3.5 trillion |
| HTTP 302 | Temporary Redirect |
| HTTP 429 | Too Many Requests |
| Quorum for 5 nodes | 3 (majority) |

---

## Trade-off Formula

> *"I'd use [X] over [Y] because [reason].
> The cost is [Z], which is acceptable here because [context]."*

---

## What Separates Good from Great

- Asks requirements before touching the design
- Draws full diagram before zooming in
- Names trade-offs explicitly — not just what, but why
- Failure thinking unprompted after every component
- Back-of-envelope estimation without hesitation
- Connects CAP choice to storage system choice
- Mentions quorum/leader election when discussing DB reliability

---

*Sessions complete: Caching, URL Shortener, Rate Limiter, Notification System, CAP, Consistent Hashing, Leader-Follower*
*Next: Pastebin / File Upload (object storage, CDN, presigned URLs)*

---

## Idempotency

**Problem:** Queues guarantee at-least-once delivery. Under failure, same message may arrive twice. Without idempotency → duplicate processing (double charge, duplicate notification).

**Definition:** An operation is idempotent if doing it N times = doing it once.

**The pattern — idempotency key:**
```
Message/request arrives with unique key
      ↓
Check Redis/DB: seen this key before?
YES → skip, return cached result
NO  → process → store key + result → return
```

**Who generates the key:** CLIENT generates it once, reuses on every retry. Never the server.

**Delivery guarantees:**
| Guarantee | Meaning | Risk |
|-----------|---------|------|
| At-most-once | Once or not at all | Can lose messages |
| At-least-once | Always delivered, may repeat | Can duplicate |
| Exactly-once | Precisely once | Complex, expensive |

**Standard pattern:** At-least-once + idempotent consumers = safe distributed processing.

**Real world:** Stripe requires idempotency key on every payment API call. Safe to retry on network failure without double charge.

**One-liner:**
> *"Client generates key once before first attempt, reuses on retry.
> Server deduplicates on it — processes once, returns cached result on repeats."*

---

## Load Balancing

**What it is:** Sits in front of servers, distributes incoming traffic across them.

**L4 vs L7:**

| | L4 (Transport) | L7 (Application) |
|---|---|---|
| Sees | IP + port only | HTTP headers, URL, cookies, body |
| Routing | Dumb — just forwards packets | Smart — content-based routing |
| Speed | Faster | Slightly slower |
| Use when | Raw TCP, max throughput | HTTP traffic, need smart routing |

**L7 example:**
```
/api/*    → API servers
/video/*  → Video servers
/static/* → CDN
Mobile User-Agent → mobile-optimized servers
```

**Routing algorithms:**
- **Round Robin** — sequential. Simple, equal servers, equal requests.
- **Weighted Round Robin** — proportional to server capacity.
- **Least Connections** — goes to server with fewest active connections. Best for variable request duration.
- **IP Hash** — same IP always → same server (sticky sessions).

**Sticky sessions:**
- Same user always hits same server
- Needed when session state is stored locally on server
- DANGEROUS — breaks horizontal scaling, single point of failure per user
- FIX: store session state in Redis → stateless servers → no sticky sessions needed

> *"Sticky sessions are a band-aid for stateful servers. The real fix is stateless servers."*

**Health checks:**
```
LB pings /health every 5s
Timeout/error → server removed from pool
Recovery → server re-added automatically
```
This enables zero-downtime deployments.

---

## All Tier 1 Concepts — Complete ✅

```
✅ Caching + invalidation
✅ Database replication + sharding
✅ CAP theorem
✅ Consistent hashing + virtual nodes
✅ Leader-follower + quorum
✅ Rate limiting (fixed/sliding window)
✅ Message queues + Kafka internals
✅ Idempotency
✅ Load balancing — L4/L7, algorithms, sticky sessions
```

*Next: Case studies — Pastebin, Feed, Chat, LMS*

---

## Circuit Breaker

**Problem it solves:** One slow/failing service causes cascading failure — threads pile up waiting, entire system chokes.

**Three states:**
```
CLOSED     → normal operation, requests flow through
OPEN       → failure threshold crossed, instant fail (no waiting)
HALF-OPEN  → cooldown expired, one test request allowed through
```

**State transitions:**
```
Failure rate > threshold → CLOSED → OPEN
Wait cooldown (30s)      → OPEN → HALF-OPEN
Test request succeeds    → HALF-OPEN → CLOSED
Test request fails       → HALF-OPEN → OPEN
```

**Why fast fail is better:** 1000 requests × 30s timeout = system choked. 1000 instant failures = system free, B gets breathing room.

**Failed messages → Dead Letter Queue (DLQ)**
- Don't drop failed messages
- Send to DLQ → retry worker processes with exponential backoff
- Backoff: 1s → 2s → 4s → 8s (don't flood recovering service)

**Graceful degradation toolkit:**
- **Circuit breaker** — stop calling failing service
- **Fallback** — return cached/default response instead of error
- **Timeout** — never wait forever, always set max wait
- **Bulkhead** — separate thread pools per downstream service

**One-liner:**
> *"CLOSED normal, OPEN fast-fail on threshold, HALF-OPEN test recovery.
> Failed messages → DLQ → retry with exponential backoff."*

---

## Object Storage + Presigned URLs + CDN

**Storage decision tree:**
```
Structured data, queryable    → PostgreSQL
Large unstructured files      → Object Storage (S3, Azure Blob)
Small key-value, fast reads   → Redis
```

**Object storage:** Store files as blobs accessed by key. No schema. Handles any size. Cheap at scale. S3, Azure Blob, GCS.

**Presigned URL:**
- Server generates temporary signed URL (~1ms, just crypto math)
- Client downloads DIRECTLY from S3 — server out of the loop
- Valid for configurable time (1hr, 24hr)
- Use for: private files, expiring content, large file downloads

```
User requests file
      ↓
Server checks DB → generates presigned URL
      ↓
Returns URL to client
      ↓
Client downloads directly from S3 (server not involved)
```

**CDN:**
- Global network of edge servers
- First request → CDN fetches from S3, caches at edge
- Subsequent requests → served from nearest edge (~5ms vs 200ms)
- Use for: public static content (images, videos, public pastes)
- Don't use for: private/expiring content (use presigned URLs instead)

**TTL alignment:**
- Paste expires at midnight → set Redis TTL to expire at midnight too
- Cache auto-invalidates when business logic says content is expired

**Pastebin pattern:**
```
text paste  → store in Postgres directly
file paste  → upload to S3, store S3 key in Postgres
public file → CDN URL
private/expiring file → presigned URL
metadata    → cache in Redis with TTL = paste expiry
```

---

## Authentication — Quick Reference

**Mental model:**
```
OAuth 2.0   = framework → "are you authorised?"  → issues access token
OIDC        = identity  → "who are you?"         → issues ID token
Bearer      = transport → how token travels in header
JWT         = format    → self-validating token format
MSAL        = library   → orchestrates all of the above
```

**The three tokens:**
| Token | Purpose | TTL | Where |
|-------|---------|-----|-------|
| Access token | Authorisation for API calls | 1hr | `Authorization: Bearer <token>` |
| ID token | User identity (email, name, groups) | 1hr | Client only, not sent to API |
| Refresh token | Get new tokens silently | 24hr+ | Browser storage |

**MSAL flow (Honeywell):**
```
Hit app → MSAL checks token → expired? → redirect to Microsoft login (OIDC)
→ get tokens → store in browser → API calls attach Bearer token automatically
→ backend validates JWT signature using Microsoft public keys
→ extract claims → check Azure AD groups → allow/deny
```

**Silent refresh:** Access token expires → MSAL intercepts 401 → uses refresh token → new access token → retry. User notices nothing.

**API auth mechanisms:**
| Mechanism | Use when |
|-----------|---------|
| Bearer JWT (OAuth) | Human users, modern APIs |
| Client Credentials | Service-to-service, background jobs |
| API Key | Third-party developer access |
| Basic Auth | Internal tools, scripts (HTTPS only) |
| mTLS | High-security microservice mesh |
| Session Cookie | Legacy, simple web apps |

---

## Kong — API Gateway

**Facade pattern:** Single entry point hiding all internal services. Client knows one URL, Kong routes internally.

```
Reverse Proxy < Load Balancer < API Gateway (Kong)
```

**Kong does:** JWT validation, rate limiting, routing, request transforms, central logging, analytics — all in one place before requests reach AKS.

**Two routing layers:**
```
Kong          → which cluster?    (outer, cross-cutting concerns)
Nginx Ingress → which service?    (inner, cluster-level)
K8s Service   → which pod?        (innermost)
```

**Why facade matters:** Backend never exposed. Single place to patch auth/rate-limit. Change backend without clients knowing.

**Full Honeywell path:**
```
Device → DNS → Firewall → Kong → AKS → Nginx Ingress → Pod
```

---

## Proxies

**Forward proxy** = in front of client → hides client from server
- Corporate filtering, VPN, anonymity
- Client configures it

**Reverse proxy** = in front of server → hides server from client
- Load balancing, TLS termination, routing, security
- Server configures it, client never knows

**DNS vs proxy:**
DNS → resolves domain to IP (before connection)
Proxy → routes request to correct backend (after connection)

---

## HTTP Versions

```
HTTP/1.0  → new TCP connection per request (slow)
HTTP/1.1  → persistent connections (keep-alive), sequential requests
            HOL blocking: slow request blocks everything behind it
HTTP/2    → multiplexing (parallel streams, one TCP connection)
            fixes HTTP HOL, still has TCP HOL blocking
HTTP/3    → QUIC (UDP-based), per-stream loss recovery
            fixes TCP HOL, 0-RTT reconnect, connection migration
```

**Persistent connections:** REST APIs use these by default. Multiple requests reuse same TCP connection transparently.

**WebSockets vs HTTP:**
- HTTP = client always initiates, request→response
- WebSocket = bidirectional, either side sends anytime
- Upgrade: `101 Switching Protocols`

---

## TLS / SSL

```
TLS solves: encryption (nobody reads data) + authentication (real server)
Uses both:  asymmetric (key exchange) + symmetric (actual data)
Certificate: domain + public key + CA signature + expiry
CA chain:   browser trusts Root CA → trusts everything it signed
```

**TLS 1.2 handshake:** 2 RTT — ClientHello → Certificate → Key Exchange → Finished

**TLS 1.3:** 1 RTT, 0-RTT reconnect, forward secrecy mandatory, weak ciphers removed

**Forward secrecy:** ephemeral session keys — private key stolen later can't decrypt past sessions

**TLS termination:** Kong handles HTTPS → forwards plain HTTP internally to pods
- Centralised cert management
- App pods do business logic only

---

## Change Data Capture (CDC)

**What:** Streams every DB change (INSERT/UPDATE/DELETE) by reading the write-ahead log (WAL)

**vs app event publishing:**
- App publishing = save to DB + emit event → dual write problem (one can fail)
- CDC = reads WAL directly → atomic, no dual write problem

**Use for:** DB → Elasticsearch sync, cache invalidation, microservice data sync, audit trail

**Stack:** Debezium + Kafka (most common)

**vs ELK logs:** ELK = app stdout logs for debugging. CDC = DB change stream for data sync. Different sources, different purposes.

---

## EDA Quick Reference

```
EDA vs REST:    REST = synchronous, both services must be up
                EDA  = async, producer emits and forgets, decoupled in time

Event          = something that happened (past, immutable fact)
Command        = instruction to do something (future, can be rejected)
Message        = umbrella term for both

Loose coupling = services don't know each other exist
                 add new consumer without touching producer

Queue vs Broker:
  Queue  = 1 message → 1 consumer → deleted (Redis, RabbitMQ)
  Broker = 1 event → N consumers → retained (Kafka, Event Hub)

Event sourcing = event log IS source of truth, state = replay of events
                 vs normal = DB is source of truth, events are side effects

Kafka offset model:
  Each consumer group has independent offset
  One group crashing doesn't affect others
  On restart: resumes from last committed offset
  Must be idempotent — same message may be processed twice
```

---

## API Gateway Landscape

```
Kong        → self-hosted, open source, plugin ecosystem, K8s (Honeywell)
AWS API GW  → managed, AWS ecosystem, Lambda integration
Azure APIM  → managed, Azure ecosystem, developer portal, deep AD integration
Apigee      → GCP, enterprise analytics
Traefik     → K8s-native, auto-discovers services
Envoy       → service mesh data plane (Istio)

North-South = external → your services → API Gateway
East-West   = service → service (internal) → Service Mesh (Istio)
```

---

## WSGI / ASGI / Gunicorn — Quick Reference

```
WSGI  = sync contract between server and Python framework
ASGI  = async successor — WebSockets, long-lived connections
Gunicorn = WSGI server (manages worker processes)
Uvicorn  = ASGI server (async)
Django's runserver = dev-only, not production
```
```
Browser → Nginx (TLS, static) → Gunicorn (workers) → application() [WSGI contract] → Django
```
Standard CRUD app → WSGI + Gunicorn sufficient. WebSockets/real-time → need ASGI.

---

## Threads vs Workers vs CPU Cores (GIL)

```
CPU core = hardware, real parallel execution limit
Worker   = process, own memory, own GIL, TRUE parallelism (Gunicorn --workers)
Thread   = inside a process, shares memory, GIL-limited

GIL: only 1 thread/process executes Python bytecode at once
  Released during I/O wait → threads help for I/O-bound work
  Never released during CPU work → threads useless for CPU-bound

CPU-bound → ProcessPoolExecutor or Celery, NOT ThreadPoolExecutor
I/O-bound → ThreadPoolExecutor is correct
```

Gunicorn sizing:
```
I/O-bound: workers = (2×cores)+1
CPU-bound: workers ≈ cores (no 2x multiplier — no wait time to hide behind)
```

---

## Why Redis Is Fast (4 Secrets)

```
1. In-memory     → RAM ~100ns vs disk ~10ms (100,000x faster)
2. Single-thread → no locks, no race conditions, atomic by default
3. epoll/kqueue  → one thread monitors thousands of sockets, processes
                   only ready ones — network I/O is the real bottleneck,
                   not command execution (microseconds vs ms round trip)
4. Optimized C structures + selective threading (6.0+ I/O threads,
                   command execution still single-threaded)

Caveat: KEYS * blocks the ENTIRE server (no second thread to pick up slack)
```

---

## How It All Connects — Throughput Across Layers

```
Throughput inversely proportional to "real work per request":
  CDN/LB/Redis  → near-zero work → millions / 100K+ RPS
  App Server    → real business logic → hundreds-low-thousands RPS
  Database      → heaviest work → 5,000-10,000 QPS

"100K RPS per server" is real for CDN/LB/Redis — NEVER real for an
app server doing actual business logic. "System handles 100K RPS"
= fleet aggregate (many app instances), not one box.

High throughput = minimize work per request (cache/CDN) +
                   parallelize what remains (horizontal scaling)
NOT: make one layer impossibly fast.
```

---

## ThreadPoolExecutor Sizing — Quick Rules

```
Single request: max_workers ≤ actual task count, bounded by downstream
                resource (DB connections, external API limits)

Multi-request worst case:
  max_workers_per_pool ≈ (available_db_connections - reserve) / gunicorn_workers

Same pattern as Rate Limiter case study — centralize state (Redis)
to enforce a real ceiling instead of trusting static worst-case math

Bounded pool at max → graceful queue, no crash
Unbounded threads at max → RuntimeError, OS refuses creation
```

---

## CONN_MAX_AGE — Don't Confuse With Capacity

```
CONN_MAX_AGE=600 → 600 SECONDS (time DB connection stays open/reused)
NOT a count of connections, NOT a request limit, NOT thread pool size

Concurrent DB capacity = workers × threads (roughly), bounded by
Postgres max_connections (default ~100)
```

---

## Prometheus + Grafana — Quick Reference

```
Prometheus = pull-based metrics collector + time-series DB + alerting
Grafana    = visualization layer reading from Prometheus (and Postgres, ELK)

Prometheus scrapes GET /metrics every 15s from:
  - Your Django pods (django-prometheus library)
  - Celery workers (celery-exporter)
  - node_exporter (CPU, memory, disk per node)
  - kube-state-metrics (pod restarts, replica counts)
```

**Four metric types:**
```
Counter   → only goes up → use rate() to get per-second rate
Gauge     → up and down → read directly (queue depth, memory)
Histogram → distributions → use histogram_quantile() for p99 latency
Summary   → pre-calculated quantiles
```

**Key PromQL patterns:**
```promql
rate(http_requests_total[5m])                              # req/sec
celery_queue_depth{queue="product_mapper"}                 # instant value
sum(rate(http_requests_total[5m])) by (endpoint)           # aggregate
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) # p99
```

**Alertmanager:** evaluates alert rules → routes to Slack/PagerDuty/Email

**Grafana data sources:** Prometheus (metrics) + Postgres (business data) + ES (logs) — all in one dashboard

**Auto-discovery in K8s:**
```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/port: "8000"
```
Prometheus finds new pods automatically — no manual config when HPA scales.

**FOC key alerts:**
```
celery_queue_depth > 1000 for 5m       → queue drift
product_mapper last completion > 6hrs  → job stuck
pod_restarts_total > 3 in 10m         → stability issue
http p99 latency > 2s                  → degradation
```

---

## Kong vs Nginx Ingress vs K8s Service

```
Kong (outside AKS):
  → SSL/TLS termination
  → JWT validation (rejects before hitting cluster)
  → Rate limiting
  → Routes to correct AKS service (upstream round-robin)
  → Knows about: services, NOT pods

Nginx Ingress (inside AKS, runs as a pod):
  → Path/host routing to correct K8s Service
  → ONE Azure LB → Nginx → infinite routing rules (cost efficient)
  → Does NOT balance across pods

K8s Service (innermost):
  → Stable DNS endpoint for pod group
  → kube-proxy round-robin across healthy pods
  → Auto-adds/removes pods as HPA scales

Kong → "Which cluster? Is it allowed?"
Nginx → "Which service inside?"
K8s Service → "Which pod?"
```

---

## FOC Fault Tolerance — Quick Reference

```
Idempotency (CJ1): UPSERT not INSERT
  ON CONFLICT (gateway_id) DO UPDATE SET ...
  Same message twice = same result

CJ2 partial failure fix:
  Staging table → process all products → atomic swap to production
  OR per-product sync timestamp → makes partial failure visible

Service Bus DLQ: failed messages retained → processed on recovery
Circuit breaker: skip failing product API, continue others, alert
Redis down: cache-aside falls through to DB (read replica absorbs)
```

---

## Real-Time Dashboard — Options

```
Frontend polling:   setInterval → constant load even when nothing changed
SSE (recommended):  server pushes when data changes, HTTP-native,
                    auto-reconnects, perfect for read-only dashboards
WebSockets:         bidirectional — overkill for display-only dashboard
Redis pub/sub:      CJ updates DB → publishes to Redis → SSE endpoint
                    pushes to browsers → ~1-2s end-to-end latency
```

---

## Product Mapper Improvement — Before/After

```
Before: 5hr data age, full sweep 200K gateways, burst queue
After:  30min data age, decoupled publisher/consumer, scalable consumers
Ideal:  5min (differential polling with modified_after filter)
        OR seconds (webhooks/CDC from product source)

Key question: does product source API support modified_after filter?
If yes → process only changed gateways → job takes minutes not hours
```
