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
- [ ] CDN — how it works, when it helps, when it doesn't
- [x] Database replication — primary-replica, read replicas, replication lag ✅ Session 1
- [x] Database sharding — strategies (hash, range), hot spots, cross-shard queries ✅ Session 1

#### Storage
- [ ] SQL vs NoSQL — when does eventual consistency become acceptable? When does ACID matter?
- [ ] Indexing — B-tree intuition, composite indexes, index trade-offs on writes
- [ ] Write-ahead logging — what it is, why it enables durability
- [ ] Object vs block vs file storage — S3 vs EBS vs EFS, when to use what

#### Communication
- [ ] REST design — idempotency, status codes, versioning
- [x] Async via queues — Kafka pub/sub, consumer groups, partitions, at-least-once ✅ Session 3
- [ ] WebSockets vs polling vs SSE — when each fits
- [ ] gRPC — why faster, when to prefer over REST

#### Reliability
- [x] CAP theorem — CP vs AP, strong vs eventual consistency ✅ Session 4
- [x] Rate limiting — fixed window, sliding window, token bucket, distributed ✅ Session 2
- [ ] Circuit breaker — what failure it prevents, how it works
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
| 4 | Pastebin / File Upload | Object storage, CDN, presigned URLs | 🔴 | Next |
| 5 | Twitter/Instagram Feed | Fan-out on write vs read | 🔴 | |
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
