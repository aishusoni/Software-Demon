# 🗄️ DB & INFRASTRUCTURE DECISIONS
> When to use what — the complete decision framework
> Reference this whenever picking a storage, queue, or search system in a design

---

## The One Rule

> **Start with PostgreSQL. Only move to something else when you have a specific reason —
> scale, schema flexibility, search, time-series, or file storage.
> Don't add complexity without justification.**

Most systems can be built with: **Postgres + Redis + S3 + Kafka**. Everything else is specialisation.

---

## Quick Decision Tree

```
What are you storing?
│
├── Structured data with relationships → PostgreSQL (default)
│     └── Need ACID across services?    → PostgreSQL
│     └── Massive write scale?          → Shard Postgres or Cassandra
│
├── Simple key-value, ultra-fast        → Redis
│     └── Need persistence + infinite scale? → DynamoDB
│
├── Flexible schema, document-shaped    → MongoDB
│
├── Massive writes, time-ordered        → Cassandra
│     └── Metrics/IoT/time-series?      → InfluxDB / TimescaleDB
│
├── Full-text search                    → Elasticsearch (alongside primary DB)
│
├── Large files/blobs                   → Object Storage (S3 / Azure Blob)
│
└── Graph relationships                 → Neo4j

What's your messaging need?
│
├── Simple background jobs (Python)     → Celery + Redis
├── High throughput, retention, multi-consumer → Kafka / Azure Event Hub
├── Reliable point-to-point, DLQ built in → Azure Service Bus / RabbitMQ
└── IoT device ingestion at scale       → IoT Hub → Event Hub
```

---

## 1. Relational Databases (SQL)

**Use when:** Relationships between entities, ACID transactions, complex queries, stable schema.

### PostgreSQL ✅ Default choice
- Most feature-rich open source SQL DB
- JSON column support (semi-structured data)
- Full ACID, excellent indexing
- **Use for:** User data, orders, payments, content metadata, most web applications
- **In our systems:** Pastebin metadata, URL shortener mappings

### MySQL
- Simpler than Postgres, historically faster for pure reads
- Massive ecosystem, widely hosted
- **Use for:** Same as Postgres — legacy systems often use MySQL
- **vs Postgres:** Postgres has better JSON, better standards compliance, more features. New projects → prefer Postgres.

### When SQL breaks down
- Millions of writes/second — single primary can't keep up
- Dynamic/flexible schema — columns change per row
- Horizontal write scaling — SQL sharding is painful
- Storing large unstructured blobs

---

## 2. NoSQL Databases

### 2a. Key-Value Stores

#### Redis
- **Built for:** In-memory ultra-fast operations
- **Latency:** ~1ms reads
- **Data structures:** Strings, lists, sets, sorted sets, hashes
- **Use for:** Caching, session storage, rate limiting, leaderboards, pub/sub, Celery broker
- **Don't use for:** Primary data store — in-memory means data loss risk on crash
- **CAP:** CP

#### DynamoDB (AWS)
- **Built for:** Massive scale key-value + document, fully managed
- **Latency:** Single-digit milliseconds
- **Use for:** Simple access patterns at massive scale — gaming, IoT device state, shopping carts, user sessions
- **Don't use for:** Complex queries, joins, aggregations — must know access patterns upfront
- **CAP:** AP (eventual default, strong consistency optional)

---

### 2b. Document Stores

#### MongoDB
- **Built for:** Flexible schema, JSON-like documents
- **Use for:** Content management, product catalogs, user profiles with varying attributes, rapid prototyping
- **Don't use for:** Joins (poor support), financial ACID data, complex transactions
- **CAP:** CP by default

---

### 2c. Wide-Column Stores

#### Cassandra
- **Built for:** Massive write throughput, high availability, geo-distributed data
- **Strengths:** Linear horizontal scaling, no single point of failure, multi-region replication
- **Use for:** Time-series data, IoT sensor readings, activity feeds, messaging history, anything with massive write volume
- **Don't use for:** ACID transactions, complex queries, low write volume (complexity not worth it)
- **CAP:** AP — tunable consistency per query
- **Key rule:** Design tables around your queries — no joins, denormalization required
- **Your context:** Honeywell IoT data at scale = Cassandra territory

---

### 2d. Graph Databases

#### Neo4j
- **Built for:** Data where relationships are the primary concern
- **Use for:** Social networks (friends of friends), recommendation engines, fraud detection, knowledge graphs
- **Don't use for:** Standard CRUD — overkill

---

## 3. Search Engines

### Elasticsearch (ELK Stack)
- **Built for:** Full-text search, log aggregation, analytics
- **Strengths:** Inverted index — blazing fast text search, fuzzy matching, relevance scoring
- **Use for:**
  - Search bars and typeahead
  - Log aggregation (Logstash → Elasticsearch → Kibana)
  - Analytics on unstructured text
- **Don't use for:** Primary DB — eventually consistent, not ACID
- **Pattern:** Write to Postgres → sync to Elasticsearch → search queries hit ES, everything else hits Postgres
- **CAP:** AP — eventual consistency

---

## 4. Time-Series Databases

### InfluxDB / TimescaleDB / Prometheus
- **Built for:** Data with timestamp as primary dimension
- **Strengths:** Optimised time-range queries, automatic downsampling, high write throughput for sequential data
- **Use for:** IoT sensor data, application metrics, financial ticks, anything queried by time range
- **Why not Postgres:** Degrades at millions of timestamped rows — special indexes + compression needed
- **Your context:** Honeywell building intelligence metrics over time = exactly this domain

---

## 5. Object Storage

### AWS S3 / Azure Blob Storage / Google Cloud Storage
- **Built for:** Large unstructured files
- **Strengths:** Infinite scale, cheap per GB, CDN integration, presigned URLs, versioning
- **Use for:** Any file > ~100KB that isn't structured data — images, videos, PDFs, ML datasets, DB backups
- **Don't use for:** Structured queryable data — use a DB
- **Presigned URLs:** Temporary signed URL — client downloads directly from storage, server out of the loop
- **Choosing between them:** Pick based on your cloud. Azure ecosystem → Blob Storage (your Honeywell context). AWS → S3. GCP → Cloud Storage. Functionally identical.

---

## 6. Message Queues & Event Streaming

### Kafka
- **Built for:** High-throughput event streaming, durable message log
- **Strengths:** Millions of messages/second, message retention + replay, multiple independent consumer groups, ordering per partition
- **Use for:** Event-driven architecture at scale, audit logs, real-time pipelines, notification fan-out, microservices at high volume
- **Don't use for:** Simple background jobs — operational overhead not worth it
- **Key property:** Messages retained after consumption — multiple consumer groups read independently

### Azure Event Hub
- **Built for:** Kafka equivalent, fully managed by Azure
- **Strengths:** Kafka-compatible API, no ops, auto-scaling, deep Azure integration
- **Use for:** Azure ecosystem + want Kafka without managing Kafka
- **Your context:** IoT Hub → Event Hub pipeline at Honeywell = exactly this
- **vs Kafka:** Functionally near-identical. Event Hub = managed. Kafka = more control.

### Azure Service Bus
- **Built for:** Enterprise messaging — reliable delivery of individual messages
- **Strengths:** DLQ built in, message sessions (ordering), transactions, scheduled messages, retry policies
- **Use for:** Point-to-point work queues, reliable ordered processing, enterprise integration
- **vs Event Hub:** Service Bus = reliable delivery of individual messages (like RabbitMQ). Event Hub = high-throughput streaming (like Kafka)
- **Your context:** IoT Hub → Event Hub → Service Bus — stream from IoT, route specific messages reliably via Service Bus ✅

### RabbitMQ
- **Built for:** Traditional message broker — queues, exchanges, routing
- **Strengths:** Flexible routing, multiple protocols, mature
- **Use for:** Complex routing, lower volume, on-premise setups
- **vs Kafka:** RabbitMQ deletes on consumption. Kafka retains. RabbitMQ = task queue. Kafka = event log.

### Celery + Redis
- **Built for:** Background task processing in Python apps
- **Use for:** Django/Flask background jobs, periodic tasks, simple async processing
- **Don't use for:** Multi-language systems, high throughput, message retention needed

---

## 7. CAP Alignment

| Database | CAP | Consistency |
|----------|-----|-------------|
| PostgreSQL | CP | Strong |
| MySQL | CP | Strong |
| Redis | CP | Strong |
| DynamoDB | AP | Eventual (strong optional) |
| Cassandra | AP | Tunable per query |
| MongoDB | CP | Strong |
| Elasticsearch | AP | Eventual |
| Kafka | AP | Eventual |

---

## 8. Interview Justification Formula

Never just say "I'd use Cassandra." Always say:

> *"I'd use [X] because [specific property it has].
> The trade-off is [what you give up],
> which is acceptable here because [why it doesn't matter for this use case]."*

**Examples:**

> *"I'd use Cassandra for the activity feed — it's AP with tunable consistency and scales writes linearly. The trade-off is no joins and eventual consistency, which is acceptable because feeds can tolerate slightly stale data and we never need cross-user queries."*

> *"I'd use PostgreSQL for payments — full ACID, strong consistency, complex queries supported. The trade-off is limited horizontal write scale, acceptable because payment volume is moderate and correctness is non-negotiable."*

> *"I'd use Elasticsearch alongside Postgres for search — inverted index gives sub-second full-text search. Postgres alone with LIKE queries would be too slow at scale. ES is eventually consistent but for search that's fine."*

---

## 9. Common Combinations

| System Type | Stack |
|-------------|-------|
| Standard web app | Postgres + Redis + S3 |
| High-scale social | Cassandra + Redis + Kafka + S3 |
| Search-heavy | Postgres + Elasticsearch + Redis |
| IoT / metrics | IoT Hub + Event Hub + TimescaleDB + Blob |
| Ed-tech platform | Postgres + Redis + S3 + Kafka + Elasticsearch |

---

## Database Normalization — 1NF Through 5NF

Using one example throughout: University Course Enrollment.

### Starting Point — Unnormalized (UNF)
```
StudentID:1, StudentName:Aish, Courses:[(Math,ProfA,Room101),(Physics,ProfB,Room102)]
```
One row holds a list of courses — nested, repeating.

### 1NF — Atomic values, no repeating groups
Flatten — one row per (student, course):

| StudentID | StudentName | CourseID | CourseName | InstructorID | InstructorOffice | Grade |
|---|---|---|---|---|---|---|
| 1 | Aish | C1 | Math | I1 | Room101 | A |
| 1 | Aish | C2 | Physics | I2 | Room102 | B |

PK: (StudentID, CourseID). Every cell is atomic. But heavy redundancy.

### 2NF — No partial dependency (non-key attr must depend on WHOLE composite key)
```
StudentName     → depends on StudentID alone     ❌ partial
CourseName      → depends on CourseID alone      ❌ partial
InstructorID    → depends on CourseID alone      ❌ partial
Grade           → depends on (StudentID+CourseID) ✅ correct
```
Decompose:
```
Student(StudentID, StudentName)
Course(CourseID, CourseName, InstructorID, InstructorOffice)
Enrollment(StudentID, CourseID, Grade)
```

### 3NF — No transitive dependency (non-key attr depends directly on key, not via another non-key)
In Course table: `CourseID → InstructorID → InstructorOffice` — two hops, not one.
Problem: if instructor changes office, must update every course row → inconsistency risk.
Decompose:
```
Course(CourseID, CourseName, InstructorID)
Instructor(InstructorID, InstructorOffice)
```

### BCNF — Every determinant must be a candidate key
3NF only checks non-key attrs. BCNF is stricter — applies to ALL functional dependencies.

Section(StudentID, CourseID, Instructor) — two candidate keys exist:
(StudentID, CourseID) and (StudentID, Instructor)

But: `Instructor → CourseID` exists, and {Instructor} alone is NOT a candidate key → BCNF violated.
Result: "ProfA teaches Math" stored in every student row — redundancy.

Decompose:
```
InstructorCourse(Instructor, CourseID)    ← ProfA→Math stored ONCE
StudentInstructor(StudentID, Instructor)
```

### 4NF — No multivalued dependency (independent multivalued facts in same table = cartesian product)
Course has: 2 instructors (ProfA, ProfB) and 2 books (Book1, Book2), completely independent.

CourseInstructorBook table forces 4 rows (2×2 combinations):
```
Math | ProfA | Book1
Math | ProfA | Book2   ← ProfA-Book2 relationship is MANUFACTURED, not real
Math | ProfB | Book1
Math | ProfB | Book2
```
Decompose:
```
CourseInstructor(CourseID, Instructor)
CourseBook(CourseID, Book)
```
2+2=4 rows, no spurious combinations.

### 5NF — No join dependency (ternary relationships needing 3-way join)
Business rule: "Instructor uses Book for Course only if: they teach Course AND Course uses Book AND Instructor has used Book before."

Three independent pairwise tables:
```
Teaches(Instructor, Course)
Uses(Course, Book)
InstructorBook(Instructor, Book)
```
Joining only two tables produces spurious rows. All three required for correct result. This IS 5NF.
Rare in practice — only appears with true three-way business constraints.

### Summary
| Form | Fixes | Rule |
|------|-------|------|
| 1NF | Repeating groups | Atomic values only |
| 2NF | Partial dependency | Non-key attrs depend on whole composite key |
| 3NF | Transitive dependency | Non-key attrs depend directly on key |
| BCNF | Non-candidate-key determinants | Every determinant is a candidate key |
| 4NF | Multivalued dependency | Independent multivalued facts → separate tables |
| 5NF | Join dependency | Ternary relationships → 3 separate tables |

Production: target 3NF/BCNF. Deliberate denormalization is common for read performance.
