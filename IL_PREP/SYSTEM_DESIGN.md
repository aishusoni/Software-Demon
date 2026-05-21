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
   - Functional: what does the system do?
   - Non-functional: scale, latency, availability, consistency needs?
   - Who are the users? Read-heavy or write-heavy?

2. ESTIMATE SCALE (back of envelope)
   - DAU → QPS → storage → bandwidth
   - Example: 10M DAU × 5 req/day = ~580 QPS
     At 1KB/req = 50GB/day writes

3. HIGH-LEVEL DESIGN
   - Draw the boxes: client → LB → service → cache → DB
   - Happy path first

4. DEEP DIVE ON CRITICAL COMPONENTS
   - Where will it bottleneck?
   - How do you scale that specific layer?

5. TRADE-OFFS
   - What did you sacrifice? Why is that acceptable here?
   - How would this evolve at 10x scale?
```

---

## Topic Map

### 🔴 Tier 1 — Non-negotiable

#### Scalability Primitives
- [ ] Horizontal scaling + statelessness — why statelessness is a prerequisite to scaling out
- [ ] Load balancing — algorithms (round robin, least conn, IP hash), L4 vs L7, sticky sessions
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
- [ ] Async via queues — why they exist, Kafka vs RabbitMQ conceptually, at-least-once vs exactly-once
- [ ] WebSockets vs polling vs SSE — when each fits
- [ ] gRPC — why faster, when to prefer over REST

#### Reliability
- [ ] CAP theorem — applying it, not just reciting it
- [ ] Rate limiting — token bucket vs leaky bucket, distributed rate limiting
- [ ] Circuit breaker — what failure it prevents, how it works
- [ ] Idempotency — why it matters in distributed systems, how to implement
- [ ] Graceful degradation — designing for partial failure

---

### 🟡 Tier 2 — Expected at strong mid-level

- [ ] Consistent hashing — what problem it solves (distributed caches, avoiding reshuffling)
- [ ] Distributed locking — when you need it, how it's done (Redis SETNX / Redlock)
- [ ] Two-phase commit vs SAGA — distributed transactions
- [ ] Event sourcing + CQRS — what they solve
- [ ] Search systems — inverted index, why LIKE queries don't scale
- [ ] Time-series data — why regular SQL struggles
- [ ] Data pipelines — batch vs stream, Lambda vs Kappa architecture

---

### 🟢 Tier 3 — Shows depth

- [ ] Consensus — Raft/Paxos intuition (leader election, quorum)
- [ ] Vector clocks — causality in distributed systems
- [ ] Bloom filters — probabilistic membership, use cases
- [ ] LSM trees vs B-trees — write-optimized vs read-optimized
- [ ] Column-oriented databases — why fast for analytics

---

## Case Studies

| # | System | Core Concepts Practiced | Status | Notes |
|---|--------|------------------------|--------|-------|
| 1 | URL Shortener | Base62, cache-aside, read replicas, sharding | ✅ Done | See session log below |
| 2 | Rate Limiter | Distributed state, token bucket, Redis | 🔴 | Next |
| 3 | Notification System | Async queues, fan-out, push vs pull | 🔴 | |
| 4 | Pastebin / File Upload | Object storage, CDN, presigned URLs | 🔴 | |
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
- Problem-first thinking — always starts with why
- Core caching insight landed cleanly
- Cache placement in URL shortener — correct and unprompted
- Read replica vs shard distinction — nailed the one-liner

**What to keep sharpening:**
- Name precision: cache-aside (not cache-head/cache-ahead), vertical vs horizontal
- Always clarify requirements before designing — skipped this step today
- State trade-offs explicitly: not just *what* you'd use but *why the trade-off is acceptable here*

**Key one-liners to remember:**
- Replica = same data copied across multiple DBs
- Shard = data split across multiple DBs
- Cache-aside = check cache → miss → fetch DB → populate cache → return
- Base62 of auto-increment ID = guaranteed unique, O(1) decode, no collision handling needed

---

## What Great Looks Like in the Room

- **Back-of-envelope without hesitation.** "10M DAU × 5 req/day = ~580 QPS."
- **Naming trade-offs explicitly.** Not just "I'd use Cassandra" — "I'm trading consistency for write throughput here, which is acceptable because..."
- **Scope control.** "I'll focus on the write path first, come back to reads."
- **Failure thinking.** After designing: what happens when this component goes down? At 10x load?

---

## Running Notes & Patterns

- URL shortener mappings are immutable → cache TTL can be very long, staleness is a non-issue
- Read-heavy systems → replicas first, sharding only when writes become the bottleneck
- Don't over-engineer: know when NOT to add complexity
