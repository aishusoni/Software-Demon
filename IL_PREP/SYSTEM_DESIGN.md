# 🏗️ SYSTEM DESIGN
> **Status:** 🔴 Not started  
> **Target:** Confidently walk through any medium-complexity system, with ed-tech awareness

---

## Core Concepts Checklist

### Fundamentals
- [ ] Horizontal vs vertical scaling
- [ ] CAP theorem — and when to pick C vs A
- [ ] Load balancing (L4 vs L7, strategies)
- [ ] Caching (Redis, CDN, cache invalidation strategies)
- [ ] SQL vs NoSQL — and when to use what
- [ ] Database sharding, replication, indexing
- [ ] Message queues (Kafka, RabbitMQ) — when and why
- [ ] API design (REST vs GraphQL vs gRPC)
- [ ] Rate limiting
- [ ] CDN & edge delivery

### Distributed Systems
- [ ] Consistency models (eventual, strong, causal)
- [ ] Distributed transactions (2PC, SAGA)
- [ ] Service discovery
- [ ] Circuit breaker pattern
- [ ] Idempotency

### Reliability & Ops
- [ ] How to design for 99.9% vs 99.99% uptime
- [ ] Observability — metrics, logs, tracing
- [ ] Graceful degradation

---

## Design Framework (Use in Every Question)

```
1. Clarify requirements (functional + non-functional)
   - Who are the users? What scale?
   - Read-heavy or write-heavy?
   - Latency requirements?

2. Estimate scale
   - DAU, QPS, storage, bandwidth

3. High-level design
   - Draw the boxes: client → API → service → DB

4. Deep dive on critical components
   - Where will it bottleneck?
   - How do you scale that layer?

5. Trade-offs
   - What did you sacrifice? Why?
   - How would you evolve this at 10x scale?
```

---

## Case Studies

### Generic (Do these first)
- [ ] Design URL shortener (Bitly)
- [ ] Design rate limiter
- [ ] Design a notification system
- [ ] Design a news feed (Twitter/Instagram)
- [ ] Design a distributed cache (Redis)
- [ ] Design a key-value store

### Ed-Tech Specific (IL-relevant — do these second)
- [ ] **Design an adaptive learning platform**
  - Students at different levels getting personalized content
  - How do you model student progress?
  - How do you serve next-best content in real-time?
- [ ] **Design an LMS (Learning Management System)**
  - Course delivery, progress tracking, teacher dashboards
  - 18M concurrent students — how?
- [ ] **Design a real-time quiz/assessment engine**
  - High write throughput during exams
  - Instant scoring, leaderboards
- [ ] **Design a recommendation engine for educational content**
  - What signals matter? (mastery, engagement, curriculum sequence)

---

## Case Study Log

| # | System | Date | What went well | What to improve |
|---|--------|------|---------------|-----------------|
| 1 | | | | |

---

## Notes & Patterns
> Add reusable insight here as you go through cases

-
