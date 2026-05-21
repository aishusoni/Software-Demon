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

3. High level design — boxes first
   - Client → LB → Service → Cache → DB
   - Happy path only, don't over-engineer early

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

**What it is:** Faster intermediate store between your app and DB. Trades staleness for speed.

**Where it sits:**
```
User → Browser cache → CDN → App cache (Redis) → DB → DB buffer pool
```

**Eviction policies:**
- LRU — kick out least recently used (default answer)
- LFU — kick out least frequently used
- TTL — expire after fixed time

**Invalidation strategies:**

| Strategy | Write goes to | On read miss | Risk |
|----------|--------------|--------------|------|
| Cache-aside | DB, delete cache | DB → repopulate | Brief staleness |
| Write-through | DB + cache | Always hits cache | Slow writes |
| Write-behind | Cache first, DB async | Always hits cache | Data loss |

**When staleness doesn't matter:** Immutable data (URL mappings, static content) → cache aggressively, long TTL fine.

**One-liner:** Cache = working memory. DB = long term memory.

---

## Database Scaling

**Read replicas:**
- Writes → primary only
- Reads → distributed across replicas
- Replication lag = tiny window of staleness (~50-100ms)
- Use when: read-heavy system, DB read load is bottleneck

**Sharding:**
- Split data across multiple DBs (not copies — splits)
- Hash sharding: hash(key) % n = which shard
- Use when: write load is the bottleneck
- Cost: cross-shard queries become complex

**One-liner:**
- Replica = same data, multiple DBs (solves read scale)
- Shard = split data, multiple DBs (solves write scale)

**Vertical vs horizontal:**
- Vertical = bigger machine (more CPU/RAM)
- Horizontal = more machines in parallel

---

## Rate Limiting

**What it is:** Enforce max N requests per user per time window. Return HTTP 429 on breach.

**Fixed window:**
```
Reset counter every 60 seconds
Problem: 99 req at t=59s + 99 req at t=61s = 198 req in 2 seconds ❌
Memory: tiny (1 counter + 1 timestamp per user)
```

**Sliding window:**
```
Store list of timestamps, look back exactly 60s from now
Accurate — no boundary exploit ✅
Memory: 10M users × 100 timestamps × 8 bytes = 8GB ❌ expensive
```

**In practice:** Fixed window with Redis is the standard answer. Sliding window if strict accuracy required.

**Distributed problem:**
- 10 servers, each with local counter = user can make 100 req per server = bypass
- Solution: all servers share ONE central Redis
- Stateless servers + centralized state = foundation of distributed systems

**Redis failure:**
- Fail open = allow all requests (availability > strict limiting) ← most common
- Fail closed = reject all (security critical systems)
- Production: Redis Sentinel for automatic failover

---

## URL Shortener

**APIs:**
```
POST /urls        { long_url } → { short_url }
GET  /{code}      → HTTP 302 redirect to long URL
```

**Code generation:**
- Base62 encode auto-increment DB ID
- 62 chars (a-z A-Z 0-9), 7 chars = 62⁷ = 3.5 trillion codes
- Guaranteed unique, O(1) decode, no collision handling needed
- Don't use MD5 hash — collision risk

**Read flow:**
```
GET /{code}
  → Check Redis cache (HIT → 302 redirect ~1ms)
  → MISS → Base62 decode → DB lookup → populate cache → 302 redirect ~20ms
```

**Why aggressive caching works:** Mappings are immutable. Once cached, always valid. Long TTL fine.

**At scale:**
- Cache absorbs most reads (popular links hit constantly)
- Read replicas for remaining DB load
- Sharding only if write load becomes bottleneck (rare for URL shortener)

---

## Distributed Systems — Core Principle

> **Stateless servers + centralized state = scalable distributed system**

Any server can handle any request because no local state is kept. State lives in Redis / DB. This is why horizontal scaling works.

---

## Numbers to Know

| Thing | Value |
|-------|-------|
| Redis read latency | ~1ms |
| DB query latency | ~10-100ms |
| Integer/timestamp | 8 bytes |
| Short string | 100 bytes |
| Default document | 1KB |
| 62⁷ combinations | ~3.5 trillion |
| HTTP 429 | Too Many Requests |
| HTTP 302 | Temporary Redirect |

---

## Trade-off Formula (Use Every Time)

> *"I'd use [X] over [Y] because [reason].
> The cost is [Z], which is acceptable here because [context]."*

Example:
> *"I'd use fixed window over sliding window because it's 50x more memory efficient.
> The cost is potential boundary exploits, which is acceptable here because we're not a
> security-critical system and brief over-limiting is tolerable."*

---

## What Separates Good from Great

- Asks requirements before touching the design
- Names trade-offs explicitly — not just what, but why
- Does failure thinking unprompted
- Knows when NOT to add complexity
- Back-of-envelope estimation without hesitation
- Scope control — "I'll come back to that"

---

*Last updated: Session 2 — Rate Limiter*
*Next: Notification System (queues, fan-out, push vs pull)*
