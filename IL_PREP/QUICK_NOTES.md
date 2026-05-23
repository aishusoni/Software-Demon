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

**When aggressive caching is safe:** Immutable data (URL mappings) — long TTL fine, staleness impossible.

---

## Database Scaling

- **Read replicas** = same data, multiple DBs → solves read scale
- **Sharding** = split data, multiple DBs → solves write scale
- **Vertical** = bigger machine. **Horizontal** = more machines.
- Replication lag = ~50-100ms, usually acceptable

---

## Rate Limiting

**Fixed window:** 1 counter + 1 timestamp per user. Memory efficient. Boundary exploit risk.

**Sliding window:** List of timestamps per user. Accurate. 50x more memory.

**Distributed:** All servers → one central Redis. Stateless servers = any server handles any request.

**Redis failure:** Fail open (allow all) = most common. Fail closed (reject all) = security critical.

---

## Message Queues

**Point-to-point:** Each message → one consumer. Good for distributing work.

**Pub/Sub:** Each message → all subscriber groups. Good for one event, multiple systems.

**Kafka specifics:**
- Topic split into partitions → each partition assigned to one consumer in a group
- More partitions = higher parallelism ceiling
- Messages retained after consumption (unlike Redis queue)
- Multiple consumer groups read same topic independently

**Celery + Redis:**
```
Celery Beat → checks periodic tasks table → pushes to Redis (the queue)
Celery Worker → pops from Redis → executes task
Redis IS the Celery queue (Redis List under the hood)
```

**Kafka vs Celery+Redis:**
- Simple background jobs → Celery + Redis
- High throughput, multiple consumers, retention needed → Kafka

---

## URL Shortener

**APIs:**
```
POST /urls  { long_url } → { short_url }
GET  /{code}             → HTTP 302 redirect
```

**Code generation:** Base62(auto-increment ID). 62⁷ = 3.5 trillion codes. No collisions.

**Read flow:** Cache check → HIT: 302 (~1ms) | MISS: DB → populate cache → 302 (~20ms)

**Scale:** Cache first → read replicas → sharding (only if writes bottleneck)

---

## Notification System

**Three flows:**
```
Direct (1→1):     Event → Queue → Worker → Send
Fan-out (1→many): Event → Fan-out Service → N messages → Workers → Send
Aggregation:      N events → Redis counter → Periodic job → 1 batched notification
```

**Two-stage fan-out:**
```
1 event → Fan-out Service → 10M individual messages → 100 workers → 10M sends
```

**Full diagram:**
```
API Layer → Kafka "events"
      ↓
Fan-out Service
  → Kafka "notifications" (direct/fan-out)
  → Redis counter (aggregation)
      ↓
Notification Workers
  → Push (APNs/FCM) | Email (SendGrid) | SMS (Twilio)

Redis counters → Celery Beat → Batched notifications
```

**Notification batching:** Many events → Redis counter → one "X people liked your post" message. Instagram/Facebook pattern.

---

## Core Distributed Systems Principle

> **Stateless servers + centralized state = scalable distributed system**

Servers remember nothing. State lives in Redis/DB. Any server handles any request. Add servers freely.

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

---

## Trade-off Formula

> *"I'd use [X] over [Y] because [reason].
> The cost is [Z], which is acceptable here because [context]."*

---

## Patterns Seen Across Systems

- Fan-out on write = expand at write time, reads stay simple
- Pub/sub = one event, multiple independent consumers
- Immutable data = cache aggressively, long TTL
- Shared state across servers = centralize it (Redis)
- High read load = replicas. High write load = sharding.
- Don't over-engineer — know when NOT to add complexity

---

## What Separates Good from Great

- Asks requirements before touching the design
- Draws full diagram before zooming in
- Names trade-offs explicitly — not just what, but why
- Failure thinking unprompted after every component
- Back-of-envelope estimation without hesitation
- Scope control — "I'll come back to that"

---

*Sessions complete: Caching, URL Shortener, Rate Limiter, Notification System*
*Next: Pastebin / File Upload (object storage, CDN, presigned URLs)*
