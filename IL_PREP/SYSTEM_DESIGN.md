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
- [ ] Caching — where to cache (client, CDN, app layer, DB layer), eviction (LRU/LFU/TTL), cache invalidation strategies
- [ ] CDN — how it works, when it helps, when it doesn't
- [ ] Database replication — primary-replica, read replicas, replication lag and its consequences
- [ ] Database sharding — strategies (range, hash, directory), hot spots, the cross-shard query problem

#### Storage
- [ ] SQL vs NoSQL — not "SQL is structured." *When* does eventual consistency become acceptable? When does ACID matter?
- [ ] Indexing — B-tree intuition, how a DB uses indexes, composite indexes, index trade-offs on writes
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
- [ ] Two-phase commit vs SAGA — distributed transactions, why 2PC is dangerous, SAGA choreography vs orchestration
- [ ] Event sourcing + CQRS — what they solve, not just what they are
- [ ] Search systems — how Elasticsearch works, inverted index, why you can't just use LIKE queries
- [ ] Time-series data — why regular SQL struggles, what you'd use (InfluxDB, TimescaleDB)
- [ ] Data pipelines — batch vs stream (Spark vs Flink conceptually), Lambda vs Kappa architecture

---

### 🟢 Tier 3 — Shows depth, good to have

- [ ] Consensus — Raft/Paxos at an intuitive level (leader election, quorum)
- [ ] Vector clocks — how distributed systems track causality
- [ ] Bloom filters — probabilistic membership, false positives, use cases
- [ ] LSM trees vs B-trees — storage engine internals, write-optimized vs read-optimized
- [ ] Column-oriented databases — why they're fast for analytics (Redshift, BigQuery)

---

## Case Studies (Do in Order)

| # | System | Core Concepts Practiced | Status | Date | Notes |
|---|--------|------------------------|--------|------|-------|
| 1 | URL Shortener | DB design, hashing, redirect at scale | 🔴 | | |
| 2 | Rate Limiter | Distributed state, token bucket, Redis | 🔴 | | |
| 3 | Notification System | Async queues, fan-out, push vs pull | 🔴 | | |
| 4 | Pastebin / File Upload | Object storage, CDN, presigned URLs | 🔴 | | |
| 5 | Twitter/Instagram Feed | Fan-out on write vs read, the core social trade-off | 🔴 | | |
| 6 | Chat System (WhatsApp) | WebSockets, message ordering, delivery guarantees | 🔴 | | |
| 7 | Typeahead / Autocomplete | Trie, caching hot queries | 🔴 | | |
| 8 | Distributed Cache (Redis) | Consistent hashing, replication | 🔴 | | |
| 9 | Job Scheduler | Distributed locking, at-least-once delivery | 🔴 | | |
| 10 | Learning Management System | Ed-tech specific, IL-relevant | 🔴 | | |

---

## What Great Looks Like in the Room

- **Back-of-envelope without hesitation.** "10M DAU × 5 req/day = ~580 QPS. At 1KB/req = 50GB/day." Practice until automatic.
- **Naming trade-offs explicitly.** Not just "I'd use Cassandra" — "I'm trading consistency for write throughput here, which is acceptable because reads can tolerate stale data."
- **Scope control.** "I'll focus on the write path first, come back to reads." Shows maturity.
- **Failure thinking.** After designing: what happens when this component goes down? At 10x load? Great candidates do this unprompted.

---

## Session Log

| Session | Topic | What clicked | What to revisit |
|---------|-------|-------------|-----------------|
| — | — | — | — |

---

## Running Notes & Patterns
> Add reusable insight here as you go through cases

-
