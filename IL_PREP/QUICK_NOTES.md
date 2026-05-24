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
