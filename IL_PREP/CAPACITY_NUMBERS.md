# 📊 CAPACITY & SCALE NUMBERS
> Know these cold. Use them in back-of-envelope estimations.
> Every limit comes from: CPU, Memory, Disk I/O, or Network.

---

## The Mental Model

```
Every service limit comes from one of four physical constraints:
  CPU      — processing power, thread count
  Memory   — RAM available  
  Disk I/O — read/write throughput
  Network  — bandwidth in/out

When a service hits its limit, one of these is the bottleneck.
Knowing which one tells you how to scale it.
```

---

## Back-of-Envelope Formulas

```
QPS (average) = (DAU × req/user/day) / 86,400
QPS (peak)    = QPS average × 3

Storage/day   = QPS × 86,400 × message_size

Memory needed = users × data_per_user

10M DAU × 10 req/day  = ~1,200 QPS avg = ~3,500 QPS peak
100M DAU × 10 req/day = ~12,000 QPS avg = ~35,000 QPS peak
```

---

## Application Servers / Pods

### Standard Pod (2 vCPU, 4GB RAM)
```
Threaded (Django/Spring):   500-2,000 RPS
Async (Node/FastAPI):       5,000-10,000 RPS
Concurrent threads:         50-200 practical
Memory per thread:          ~1-2MB
```

### Why async is faster
No context switching. Event loop handles thousands of connections with one thread — as long as operations are non-blocking.

---

## Databases

### PostgreSQL (single primary)
```
Reads:        5,000-10,000 QPS
Writes:       1,000-3,000 QPS
Connections:  100-500 (use PgBouncer beyond this)
Storage:      TBs easily
Scale read:   Add replicas (each adds 5-10K read QPS)
Scale write:  Shard (each shard adds 1-3K write QPS)
```

### Redis (single node)
```
Reads/Writes: 100,000-1,000,000 ops/sec
Connections:  10,000+ concurrent clients
Limit:        RAM size
Latency:      ~1ms
```

### Cassandra (per node, 8 vCPU 32GB)
```
Writes:    10,000-30,000/sec  ← writes faster than reads (LSM tree)
Reads:     5,000-15,000/sec
Storage:   1-4TB per node
Scaling:   LINEAR — double nodes = double throughput
```

### DynamoDB
```
Latency:   < 10ms guaranteed
Scale:     Essentially unlimited (AWS manages it)
Use:       When you need guaranteed latency at any scale
```

### MongoDB (single node)
```
Reads:     10,000-50,000 QPS
Writes:    5,000-20,000 QPS
```

---

## Message Queues

### Kafka (single broker, 8 vCPU 32GB)
```
Throughput:  100MB/sec - 1GB/sec
Messages:    100,000-1,000,000/sec
Latency:     2-10ms
Retention:   Days to weeks (storage-bound)
Consumers:   1 per partition per consumer group
```

### Kafka (3-broker cluster, 100 partitions)
```
Messages:    500,000-5,000,000/sec
Consumers:   Up to 100 per consumer group
```

### Azure Event Hub
```
Basic:     20MB/sec ingress
Premium:   Scales to 1,000,000+ events/sec
Retention: 1-90 days
```

### Azure Service Bus
```
Standard:  ~115 ops/sec sustained
Premium:   1,000-4,000 messages/sec per unit
Max size:  256KB (Standard), 100MB (Premium)
```

### Redis Queue (Celery)
```
Enqueue:   50,000-100,000 tasks/sec
```

---

## Object Storage (S3 / Azure Blob)

```
PUT rate:   3,500/sec per prefix
GET rate:   5,500/sec per prefix
Throughput: Multiple GB/sec per prefix
Storage:    Unlimited
File size:  Up to 5TB per object
Latency:    100-200ms cold, 10-50ms warm
With CDN:   5-50ms from nearest edge
```

---

## Load Balancers

### Nginx (single instance)
```
RPS:         50,000-100,000 (static)
             10,000-50,000 (proxying)
Connections: 10,000-100,000 concurrent
```

### AWS ALB
```
RPS:         Millions (auto-scales)
Latency:     < 1ms added
```

---

## WebSockets

```
Single server (8 vCPU, 16GB):
  Connections:   10,000-100,000 concurrent
  Memory/conn:   ~2-10KB
  
K8s pod (2 vCPU, 4GB):
  Connections:   10,000-50,000 concurrent

1M concurrent connections = ~20 dedicated WebSocket servers
```

**Bottleneck:** Memory (each connection holds state), not CPU.

---

## Kubernetes

```
Pods per node:      110 (default limit)
Nodes per cluster:  5,000 (default limit)
Total pods:         550,000 max

Small node  (2 vCPU, 8GB):    10-20 pods
Medium node (8 vCPU, 32GB):   40-80 pods
Large node  (32 vCPU, 128GB): 100+ pods
```

---

## The Scale Ladder

```
Writes/sec     Action
─────────────────────────────────────────
< 1,000        Single Postgres — fine
1,000-5,000    + PgBouncer connection pooling
5,000-20,000   + Read replicas
20,000-100K    Shard Postgres or switch to Cassandra
> 100K         Cassandra / DynamoDB

Messages/sec   Action
─────────────────────────────────────────
< 1,000        Celery + Redis — fine
1,000-10K      RabbitMQ or Service Bus
10K-100K       Kafka (small cluster)
> 100K         Kafka multi-broker or Event Hub Premium

Concurrent     Action
connections
─────────────────────────────────────────
< 10K          Single app server
10K-100K       Multiple pods behind LB
100K-1M        Dedicated WebSocket servers
> 1M           Distributed WS cluster
```

---

## Interview Estimation Template

```
1. State: "10M DAU, ~50 requests/user/day"
2. Calculate: 10M × 50 / 86,400 = ~5,800 QPS avg
3. Peak: 5,800 × 3 = ~17,000 QPS peak
4. Check: Single Postgres handles ~5K writes/sec
5. Decide: "Mostly reads — I'd add 2 read replicas.
            Write volume is ~2K/sec, single primary handles it."
```

---

## Key Numbers to Memorize

```
Redis:           1,000,000 ops/sec (in-memory superpower)
Postgres:        5,000-10,000 QPS (with indexing)
Kafka:           1,000,000 messages/sec (per broker)
Nginx:           50,000-100,000 RPS
WebSocket:       100,000 connections per server
K8s node:        110 pods max
K8s cluster:     5,000 nodes max
Peak multiplier: 3x average QPS
Threads/pod:     50-200 practical
Memory/thread:   ~1-2MB
Memory/WS conn:  ~2-10KB
S3 GET rate:     5,500/sec per prefix
```
