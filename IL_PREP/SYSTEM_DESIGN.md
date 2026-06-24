# 🏗️ SYSTEM DESIGN
> **Status:** 🟡 In Progress  
> **Target:** Confidently design any medium-complexity system, articulate trade-offs, handle follow-ups

---

## What's Actually Being Tested at 3yr Level

Not scale of knowledge — quality of thinking.
- Do you ask the right questions before drawing boxes?
- Can you make a decision under ambiguity and defend it?
- Do you know *why* you'd pick X over Y, not just *that* you would?
- Can you identify where a system breaks and fix it?

**The failure mode to avoid:** Jumping to solutions before understanding the problem. Knowing components in isolation but not wiring them together under pressure.

**The two layers:**
- **Layer 1 — Building blocks:** individual components (DB, cache, queue, CDN...). Necessary but not sufficient.
- **Layer 2 — Systems thinking:** how components interact under real constraints — latency, failure, cost, consistency. This needs deliberate practice through case studies.

---

## Design Framework (Use in Every Question)

```
1. CLARIFY REQUIREMENTS (2–3 min)
   - Functional: what does it do?
   - Non-functional: scale, latency, availability, consistency needs?
   - Who are the users? Read-heavy or write-heavy?
   - What happens on failure / limit exceeded?

2. ESTIMATE SCALE (back of envelope)
   - DAU → QPS → storage → bandwidth
   - Example: 10M DAU × 5 req/day = ~580 QPS
     At 1KB/req = 50GB/day writes

3. HIGH-LEVEL DESIGN — boxes first, always
   - Client → LB → Service → Cache → DB
   - Happy path only, don't over-engineer early
   - FINISH THE BOXES before zooming into any component

4. DEEP DIVE ON CRITICAL COMPONENTS
   - Where will it bottleneck?
   - How do you scale that specific layer?

5. TRADE-OFFS
   - What did you sacrifice? Why is that acceptable here?
   - How would this evolve at 10x scale?
   - What happens when this component fails?
```

---

## Topic Map

### 🔴 Tier 1 — Non-negotiable

#### Scalability Primitives
- [x] Horizontal scaling + statelessness — why statelessness is a prerequisite to scaling out ✅ Session 4
- [x] Load balancing — L4 vs L7, algorithms (round robin, least conn, IP hash), sticky sessions ✅ Session 4
- [x] Caching — where to cache, eviction (LRU/LFU/TTL), cache invalidation strategies ✅ Session 1
- [x] CDN — edge caching, global latency, push vs pull, public vs private ✅ Session 5
- [x] Database replication — primary-replica, read replicas, replication lag ✅ Session 1
- [x] Database sharding — strategies (hash, range), hot spots, cross-shard queries ✅ Session 1

#### Storage
- [ ] SQL vs NoSQL — when does eventual consistency become acceptable? When does ACID matter?
- [ ] Indexing — B-tree intuition, composite indexes, index trade-offs on writes
- [ ] Write-ahead logging — what it is, why it enables durability
- [x] Object storage — S3/Blob, presigned URLs, unstructured files ✅ Session 5

#### Communication
- [ ] REST design — idempotency, status codes, versioning
- [x] Async via queues — Kafka pub/sub, consumer groups, partitions, at-least-once ✅ Session 3
- [ ] WebSockets vs polling vs SSE — when each fits
- [ ] gRPC — why faster, when to prefer over REST

#### Reliability
- [x] CAP theorem — CP vs AP, strong vs eventual consistency ✅ Session 4
- [x] Rate limiting — fixed window, sliding window, token bucket, distributed ✅ Session 2
- [x] Circuit breaker — CLOSED/OPEN/HALF-OPEN, DLQ, exponential backoff ✅ Session 5
- [x] Idempotency — at-least-once delivery, idempotency keys, client generates key ✅ Session 4
- [ ] Graceful degradation — designing for partial failure

---

### 🟡 Tier 2 — Expected at strong mid-level

- [x] Consistent hashing — ring model, virtual nodes, ~1/N remapping ✅ Session 4
- [ ] Distributed locking — Redis SETNX / Redlock
- [ ] Two-phase commit vs SAGA — distributed transactions
- [ ] Event sourcing + CQRS — what they solve
- [ ] Search systems — inverted index, why LIKE queries don't scale
- [ ] Time-series data — why regular SQL struggles
- [ ] Data pipelines — batch vs stream, Lambda vs Kappa architecture

---

### 🟢 Tier 3 — Shows depth

- [ ] Consensus — Raft/Paxos intuition
- [ ] Vector clocks — causality in distributed systems
- [ ] Bloom filters — probabilistic membership
- [ ] LSM trees vs B-trees — write vs read optimized
- [ ] Column-oriented databases — why fast for analytics

---

## Case Studies

| # | System | Core Concepts Practiced | Status | Notes |
|---|--------|------------------------|--------|-------|
| 1 | URL Shortener | Base62, cache-aside, read replicas, sharding | ✅ Done | |
| 2 | Rate Limiter | Fixed/sliding window, Redis central state, distributed problem | ✅ Done | |
| 3 | Notification System | Queues, fan-out, batching, pub/sub, Celery+Redis | ✅ Done | |
| 4 | Pastebin / File Upload | Object storage, CDN, presigned URLs, TTL alignment | ✅ Done | |
| 5 | Twitter/Instagram Feed | Fan-out on write vs read | 🔴 | Next |
| 6 | Chat System (WhatsApp) | WebSockets, message ordering | 🔴 | |
| 7 | Typeahead / Autocomplete | Trie, caching hot queries | 🔴 | |
| 8 | Distributed Cache (Redis) | Consistent hashing, replication | 🔴 | |
| 9 | Job Scheduler | Distributed locking, at-least-once delivery | 🔴 | |
| 10 | Learning Management System | Ed-tech specific, IL-relevant | 🔴 | |

---

## Session Log

### Session 1 — Caching + URL Shortener
**Concepts covered:** Caching (what, why, where), eviction (LRU), invalidation strategies (cache-aside, write-through, write-behind), read replicas, sharding, Base62 encoding, horizontal vs vertical scaling

**What was strong:**
- Problem-first thinking
- Core caching insight landed cleanly
- Cache placement — correct and unprompted
- Replica vs shard one-liner nailed

**What to keep sharpening:**
- Name precision: cache-aside, vertical vs horizontal
- Always clarify requirements before designing
- State trade-offs explicitly

**Key one-liners:**
- Replica = same data copied across multiple DBs
- Shard = data split across multiple DBs
- Cache-aside = check cache → miss → fetch DB → populate cache → return
- Base62 of auto-increment ID = guaranteed unique, O(1) decode, no collisions

---

### Session 2 — Rate Limiter
**Concepts covered:** Fixed window counter, sliding window timestamps, memory estimation, distributed state problem, centralizing state in Redis, Redis failure strategies

**What was strong:**
- Asked requirements questions upfront — big improvement
- Correctly identified Redis over DB
- Understood fixed window boundary exploit
- Memory estimation clean and correct
- Redis + Celery architecture explained correctly from own experience

**What to keep sharpening:**
- After designing a component: "what happens when this fails?"
- Trade-off articulation — decision → mechanism → cost → when acceptable

**Key one-liners:**
- Fixed window = memory efficient, has boundary exploit
- Sliding window = accurate, 50x more memory
- Distributed state solution = centralize in Redis, keep servers stateless
- Fail open = allow all on Redis failure (most common choice)

**Size reference:**
- Integer/timestamp = 8 bytes
- Short string = 100 bytes
- Default document = 1KB

---

### Session 3 — Notification System
**Concepts covered:** Kafka pub/sub, consumer groups, partitions, two-stage fan-out, notification batching/aggregation, Celery+Redis architecture, three notification flows

**What was strong:**
- Immediately reached for queues — correct and unprompted
- Decoupling reasoning was clear and well articulated
- Partition/consumer sizing was correct
- Independently invented notification batching — not prompted
- Connected Redis counters + periodic jobs naturally

**What to keep sharpening:**
- Draw full architecture diagram BEFORE zooming into sizing
- Finish the boxes first, then go deep

**Key patterns:**
- Fan-out on write = expand work at write time, simple reads
- Two-stage fan-out = 1 event → fan-out service → N individual messages → workers
- Notification batching = Redis counter + periodic job → one aggregated notification
- Three flows: direct (1→1), fan-out (1→many), aggregation (many→1 batched)

**Full notification system diagram:**
```
Event Sources
      ↓
API Layer → Kafka topic: "events"
      ↓
Fan-out Service
  → direct/fan-out → Kafka topic: "notifications"
  → aggregate      → Redis counter
      ↓
Notification Workers (100 consumers)
  → checks user preferences
  → routes to Push / Email / SMS service
      ↓
Push (APNs/FCM) | Email (SendGrid) | SMS (Twilio)
      ↓
End User Device

Separately:
Redis counters → Celery Beat → Aggregation job → Workers → Batched notification
```

---

## What Great Looks Like in the Room

- **Back-of-envelope without hesitation**
- **Naming trade-offs explicitly** — not just what, but why
- **Finish the diagram before zooming in** — boxes first, depth second
- **Failure thinking** — unprompted, after every component
- **Scope control** — "I'll come back to that"

---

## Running Notes & Patterns

- Stateless servers + centralized state = foundation of scalable distributed systems
- URL shortener mappings are immutable → aggressive caching, long TTL fine
- Read-heavy → replicas first. Write bottleneck → sharding
- Don't over-engineer — know when NOT to add complexity
- Fan-out on write = expand at write time, reads stay simple
- Pub/sub = one event, multiple independent consumer groups
- Kafka partitions = ceiling on parallelism. More partitions = more consumers possible
- Rate limiter Redis failure → fail open (availability > strict limiting)
- Notification batching = many events → one aggregated message (Instagram like pattern)


### Session 4 — Core Distributed Systems Concepts
**Concepts covered:** CAP theorem, consistent hashing + virtual nodes, leader-follower + quorum, idempotency, load balancing (L4/L7), statelessness, delivery guarantees

**What was strong:**
- CAP applied correctly to Instagram likes (AP) and banking (CP) — right reasoning, right answer
- Consistent hashing — explained ring model + virtual nodes fully without prompting
- Quorum — correctly identified that majority prevents two groups from simultaneously holding it
- Load balancing — L7 reasoning for video streaming was clean and immediate
- Idempotency — correctly applied idempotency key pattern to double payment scenario

**Key one-liners:**
- CAP: P is non-negotiable, real choice is C vs A when partition occurs
- Consistent hashing: ~1/N remapping vs ~100% for normal hashing
- Virtual nodes: multiple ring positions per server = even load distribution
- Leader-follower: one leader takes writes, followers replicate + serve reads
- Quorum (majority): two groups can't simultaneously hold majority → split-brain impossible
- Idempotency: client generates key once, reuses on retry, server deduplicates
- L4 = IP+port only. L7 = content-aware routing. Most web apps use L7.
- Sticky sessions = band-aid. Real fix = stateless servers + Redis for session state.

**Delivery guarantees:**
- At-most-once = can lose messages
- At-least-once = can duplicate (Kafka default) → needs idempotent consumers
- Exactly-once = expensive, complex

**All Tier 1 concepts now complete ✅**

### Session 5 — Circuit Breaker + Pastebin
**Concepts covered:** Circuit breaker (CLOSED/OPEN/HALF-OPEN), thread starvation, cascade failure, DLQ + exponential backoff, object storage, presigned URLs, CDN, TTL alignment with expiry

**What was strong:**
- Dead letter queue — named correctly and unprompted
- Blob/object storage — reached for it independently
- Cache placement — correct and unprompted
- TTL alignment to paste expiry — understood immediately

**What to keep sharpening:**
- Object storage should be first instinct for any file >1MB, not NoSQL

**Key concepts:**
- Thread starvation = all threads frozen waiting on slow dependency
- Cascade failure = starvation spreads service to service
- Circuit breaker = fast fail preserves thread pool
- Object storage = unstructured files, accessed by key, not DB rows
- Presigned URL = temporary signed URL, client downloads directly from S3, server out of loop
- CDN = edge servers globally, cache public static content close to user
- TTL alignment = set Redis TTL = paste expiry time → auto-invalidates

**Pastebin full flow:**
```
Create: POST /paste → text→Postgres | file→S3+Postgres key → Base62 ID → return link
Read:   GET /abc123 → Redis cache → MISS→Postgres
        text → return directly
        file public  → CDN URL
        file private → presigned S3 URL (expires with paste)
```

---

## Event-Driven Architecture (EDA)

### EDA vs REST — Core Distinction

```
REST (synchronous):
  Service A → POST /inventory/decrease → waits for response
  Both services must be up simultaneously. A is blocked until B responds.

EDA (asynchronous):
  Service A → emits "order.placed" event → done, moves on
  Service B, C, D each react independently when ready
  A doesn't know or care who's listening
```

**Key differentiator: decoupling in time.**
Producer emits and forgets. Consumer processes whenever ready. They never talk directly.

---

### Event vs Command vs Message

| Term | Meaning | Tense | Can be rejected? |
|------|---------|-------|-----------------|
| **Event** | Something that happened | Past — "order was placed" | No — it's a fact |
| **Command** | Instruction to do something | Future — "place this order" | Yes — can fail |
| **Message** | Umbrella term | Either | Either |

In EDA you mostly deal in events — immutable facts about what happened.

---

### Loose Coupling — What It Actually Means

Services don't know about each other. Order Service doesn't import, call, or reference Inventory Service.

```
Tight coupling (REST) breaking:
  → Inventory Service down = Order Service fails too
  → Add Email Service = must modify Order Service code

Loose coupling (EDA) benefit:
  → Inventory Service down = event waits in queue, processed on recovery
  → Add Email Service = just subscribe to topic, Order Service unchanged
```

---

### Message Queue vs Event Broker

| | Message Queue | Event Broker |
|---|---|---|
| Examples | Redis List, RabbitMQ, Service Bus | Kafka, Azure Event Hub |
| Message consumed by | ONE consumer | MANY independent consumers |
| After consumption | Deleted | Retained (configurable) |
| Use when | Simple background jobs, point-to-point | Multiple consumers, audit log, replay |

Celery + Redis = message queue pattern. Kafka = event broker pattern.

---

### Event Sourcing

**Normal system:** Store current state only. History is lost.
```
orders table: { id: 123, status: "shipped" }  ← don't know previous states
```

**Event sourcing:** Store every event that ever happened. Current state = replay of all events.
```
events log:
  order.created  → { id: 123, items: [...] }
  order.paid     → { id: 123, amount: 500 }
  order.shipped  → { id: 123, tracking: "xyz" }

Current state = derived by replaying all events
```

**Benefits:** Complete audit trail, replay history, reprocess with new logic retroactively.

**vs just publishing events:**
- Publishing events = side effect of DB write. DB is still source of truth.
- Event sourcing = event log IS the source of truth. DB state is just a projection.

Kafka naturally supports event sourcing — it retains all events.

---

### Failure Modes

**Publishing guarantees:**
- At-most-once = may lose messages (fire and forget)
- At-least-once = may duplicate (Kafka default → needs idempotent consumers)
- Exactly-once = no loss, no duplicate (expensive, Kafka transactions)

**When consumer goes down:**
```
Broker retains messages in partition/queue
Consumer restarts → resumes from last committed offset
Risk: consumer lag grows (backlog builds faster than consumed)
Solution: monitor consumer lag, scale out consumers if needed
```

**When broker goes down:**
```
Producers can't publish → need retry + backoff
Consumers halt → processing stops
Solution: broker replication (Kafka replicas, Event Hub geo-redundancy)
```

**Consumer lag** = the real day-to-day failure mode. Consumer alive but slow. Fix: add consumers (up to partition count ceiling).

---

### Multi-Consumer Failure — Kafka Offset Model

Each consumer group has its own **independent offset**. One group crashing never affects others.

```
Event: "order.placed" published at offset 100

Service 1 group offset: 100 → processing normally ✅
Service 3 group offset: 100 → processing normally ✅
Service 5 group offset: 87  → crashed, stuck here ❌

Kafka retains all messages
Service 5 restarts → resumes from offset 87
Services 1 and 3 unaffected throughout
```

**Critical:** Service 5 on restart will reprocess messages from offset 87. Some may have been partially processed before crash. Consumers MUST be idempotent — processing same message twice must be safe.

DLQ applies when a message fails after N retries — moved to dead letter queue for manual inspection. Not for crashes (offsets handle that).

---

### Honeywell IoT Hub → Event Hub → Service Bus

```
IoT Hub     = central device connectivity platform
              receives all device telemetry
              temporary store, multiple consumers can read

Event Hub   = high-throughput event streaming per product domain
              FOC Event Hub, Building Intelligence Event Hub (separate concerns)
              multiple consumer groups read independently

Service Bus = reliable point-to-point messaging
              at-least-once delivery, DLQ built in
              used for lower volume, critical messages needing guaranteed delivery
```

Why multiple Event Hubs (not one): independent scaling, independent failure domains, different products don't interfere with each other.

If Event Hub removed: IoT Hub has no consumers. Device data accumulates with nowhere to go. All downstream products lose their data feed.

---

### Kong AuthN vs App AuthZ (from Honeywell architecture)

```
Kong     = AuthN = "Is this token cryptographically valid?"
           Uses Microsoft public keys to verify JWT signature
           Rejects with 401 — never reaches AKS

Your app = AuthZ = "Is this user allowed to do THIS specific thing?"
           Reads claims (email, groups) from JWT
           Checks business permissions (tenant, role, resource)
           Rejects with 403 — valid token, wrong permissions
```

401 = Kong. 403 = your app. Two different layers, two different concerns.

---

## Change Data Capture (CDC)

### What It Is
CDC tracks every change (INSERT, UPDATE, DELETE) in your database and streams those changes to other systems in real time.

```
Database row changes
      ↓
CDC reads database write-ahead log (WAL)
      ↓
Publishes change event to Kafka:
  { op: "UPDATE", table: "orders",
    before: { status: "pending" },
    after:  { status: "shipped" } }
      ↓
Downstream systems consume (Elasticsearch, cache, other services)
```

### Why CDC Exists — Problems It Solves

**Problem 1 — Keeping systems in sync (DB → Elasticsearch)**
```
Without CDC (polling): SELECT * FROM products WHERE updated_at > last_check
  → delayed, misses deletes, hammers DB

With CDC: every change streams instantly → Elasticsearch always in sync
```

**Problem 2 — Cache invalidation**
DB row changes → CDC event → automatically invalidate Redis cache entry. No app code needed.

**Problem 3 — Microservice data sync**
Service A owns User table. Service B needs user data.
CDC streams User table changes to Kafka → Service B consumes. No direct API calls.

**Problem 4 — Audit trail**
Complete immutable history of every data change, who changed what, before/after values.

### CDC vs Application Event Publishing
```
App publishing:  Save to DB → emit event in code
  Problem: DB save succeeds, event publish fails → systems out of sync (dual write problem)

CDC:  Save to DB → CDC reads WAL → publishes event
  WAL write and CDC read are atomic — if it's in the WAL, CDC publishes it
  No dual write problem → more reliable
```

### Common CDC Tools
| Tool | Works with | Publishes to |
|------|-----------|-------------|
| Debezium | Postgres, MySQL, MongoDB | Kafka |
| AWS DMS | Most DBs | Kafka, S3, Redshift |
| Azure Data Factory | Most DBs | Event Hub, Blob |

### ELK Logs vs CDC — Not the Same
```
ELK (log aggregation):
  Source: application stdout/stderr
  Purpose: debugging, monitoring, alerting
  Format: log lines

CDC:
  Source: database WAL
  Purpose: data sync, audit trail, cache invalidation
  Format: structured change events
```

Your AKS → ELK setup is log aggregation — pods write logs, Filebeat ships to Elasticsearch, Kibana for search. CDC would be a separate pipeline tapping the DB directly.

### One-liner
> *"CDC reads the database write-ahead log and streams every data change as an event. More reliable than application-level publishing — eliminates the dual write problem. Debezium + Kafka is the standard stack."*

---

## Real-Time Dashboard Patterns

### The Problem
Polling-based dashboards create constant server load even when nothing changed. Every connected user's browser fires requests on a timer regardless of whether data changed.

### Pattern 1 — Frontend Polling (Simple, Inefficient)
```javascript
setInterval(() => fetch('/api/gateways').then(update), 30000)
// Every user × every 30s → server load scales with user count
```

### Pattern 2 — SSE (Server-Sent Events, Recommended for Dashboards)
```
Server → Client only (one-directional push)
HTTP-native — works with existing Nginx, LB, no protocol upgrade
Auto-reconnects on network drop
Perfect for: read-only dashboards, monitoring screens, live feeds

Client:
  const events = new EventSource('/api/gateway-stream')
  events.onmessage = (e) => updateDashboard(JSON.parse(e.data))

Server:
  def gateway_stream(request):
      def event_stream():
          while True:
              changes = get_recent_gateway_changes()
              if changes:
                  yield f"data: {json.dumps(changes)}\n\n"
              time.sleep(5)
      return StreamingHttpResponse(event_stream(), content_type='text/event-stream')
```

### Pattern 3 — WebSockets (Bidirectional, Overkill for Read-Only)
```
Use when: clients ALSO send data in real-time (chat, collaborative editing)
SSE is simpler and sufficient for dashboards — no need for WS
```

### Pattern 4 — Redis Pub/Sub + SSE (Production Pattern)
```
Data changes in DB
      ↓
CJ1/CJ2 publishes to Redis channel: redis.publish("gateway_updates", data)
      ↓
SSE endpoint subscribed to Redis channel
      ↓
Moment data changes → SSE pushes to ALL connected browsers simultaneously
→ ~1-2 second end-to-end latency
→ Decouples "data changed" from "browser notification"
→ Scales — Redis pub/sub handles many SSE subscribers
```

### When to Use Which
```
Read-only dashboard, updates ≤ few seconds:  SSE + Redis pub/sub
Bidirectional (user sends + receives):        WebSockets
Simple, low frequency (>30s intervals):      Frontend polling is fine
```
