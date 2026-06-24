# 🏗️ FOC ARCHITECTURE — Full Breakdown
> Forge Operation Center — current architecture, problems, better design, component alternatives
> Use this for interview discussions about your Honeywell experience

---

## 1. Current System (Reconstructed)

```
IoT Hub          → receives gateway telemetry, connect/disconnect events
                   polled every 15 mins for status

Product Sources  → external API: gateway→site→customer→region→asset→point data
                   polled every 5 hours via Celery Beat (product mapper task)

Celery + Redis   → product mapper (periodic, 5hrs)
                   decorator task → alerts if metrics drift >5%
                   email alerts for cron outcomes

Azure Blob       → raw gateway telemetry files (volume calculation)

ELK Stack        → pod logs, log-based alerts (restarts, failures)

AKS              → Django backend, Celery workers + Beat, React frontend

Kong             → API gateway (auth, routing)
MSAL             → Azure AD auth for UI users
Middleware       → bearer token / API key / basic auth for service-to-service

CI/CD            → GitHub Actions → Coverity + BlackDuck + SonarQube
                   → JFrog Artifactory → Octopus → AKS

Flower           → Celery monitoring (partially integrated)
pytest           → unit + functional tests
```

---

## 2. Celery Queue Drift — Root Cause Analysis

**Most likely: Product mapper fan-out burst**
```
Beat fires every 5hrs → product mapper creates 1 task per gateway
10,000 gateways → 10,000 tasks enqueued simultaneously
Workers process 100/min → 100 mins to drain
If processing slows to 50/min → tasks accumulate faster than drained
Next Beat fires in 5hrs → another 10,000 tasks → drift compounds
```

**Also likely: Worker memory leak → OOM kill loop**
```
Workers accumulate memory over 1000s of tasks (ORM, large payloads)
Worker pod hits K8s memory limit → OOM killed → restarts
During restart (30-60s) → tasks accumulate
Happens repeatedly → chronic drift
Fix: worker_max_tasks_per_child = 1000 in Celery config
```

**Also possible: External API rate limiting**
```
Product source API rate-limits your calls
Tasks time out → retry → re-enqueued → pile up on top of new tasks
```

**What Flower would have shown:**
```
Active workers: 5 (normal)
Queue depth:    8,432 (the smoking gun)
Task rate:      12/min (vs normal 200/min → something is slow)
Failed tasks:   943 (retry storm → external failures)
```

**Fix: HPA on queue depth**
```yaml
# Scale workers based on Redis queue depth (via Prometheus custom metric)
metrics:
- type: External
  external:
    metric:
      name: redis_queue_depth
    target:
      type: AverageValue
      averageValue: "100"    # scale up when >100 tasks per worker
```

---

## 3. Better Architecture

### Core Problem: Pull Model → Push/Event Model

```
Current: poll every 5hrs (stale, burst, drift)
Better:  react to changes + staggered refresh
```

### Proposed Design

```
IoT Hub (real-time events)
      ↓
Event Hub → Stream Processor
  → gateway status changes → Redis cache immediately (real-time)
  → new gateway detected → trigger incremental product fetch
      ↓
Staggered Product Sync (instead of all-at-once burst):
  Divide gateways into N buckets
  Bucket 1 → refresh at t=0
  Bucket 2 → refresh at t=30min
  ...
  Smooth load, queue never spikes

OR if product source has last_modified:
  Only re-fetch gateways that actually changed → 90% fewer tasks
      ↓
Data Layer:
  PostgreSQL    → gateway/site/customer/asset structure
  Redis         → hot data cache (current status, recent queries)
  TimescaleDB   → time-series metrics from gateways
  Azure Blob    → raw telemetry (keep as-is)
      ↓
API Layer:
  Django + Kong (keep as-is)
  Add: GraphQL (optional) → frontend queries exactly what it needs
  Add: WebSocket/SSE → push gateway status to UI in real-time
      ↓
Frontend: React + MSAL (keep as-is)
  Add: real-time status via WebSocket (no 15-min polling)
```

---

## 4. Kubernetes Deep Dive

### What K8s Provides
```
Self-healing:      pod crashes → auto-restarted
Auto-scaling:      HPA scales pods based on metrics
Rolling updates:   zero-downtime deployments
Service discovery: services find each other by name
Config management: ConfigMaps + Secrets
Namespaces:        isolation per tenant/environment
```

### Platforms
| Platform | Type | Best for |
|---------|------|---------|
| AKS (Azure) | Managed | Your stack — Azure ecosystem |
| EKS (AWS) | Managed | AWS ecosystem |
| GKE (Google) | Managed | Best managed K8s, autopilot |
| Minikube | Local | Testing K8s locally |
| k3s | Lightweight | Small VMs, cheap |
| Render/Railway | No K8s | Small projects, abstracted |

### Key K8s Configs for FOC

```yaml
# Prevent memory leak OOM kills — tune these numbers
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "500m"

# Graceful Celery worker shutdown (drain in-flight tasks before kill)
lifecycle:
  preStop:
    exec:
      command: ["celery", "control", "shutdown"]

# HPA on queue depth (custom metric)
spec:
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: External
    external:
      metric:
        name: redis_queue_depth
      target:
        type: AverageValue
        averageValue: "100"
```

---

## 5. Monitoring + Observability Stack

### Three Pillars
```
Logs    → what happened    → ELK (you have this)
Metrics → how it's behaving → Prometheus + Grafana (add this)
Traces  → how requests flow → Jaeger/Tempo (nice to have)
```

### Full Stack
```
AKS Pods stdout/stderr
  → Filebeat → Elasticsearch → Kibana (logs, log-based alerts)

Service /metrics endpoints
  → Prometheus scrapes every 15s → stores time-series
  → Alertmanager → Slack / PagerDuty / Email
  → Grafana → dashboards

Celery
  → Flower → /metrics → Prometheus (queue depth, worker status, task rate)
```

### Key Metrics to Track
```
celery_queue_depth{queue="product_mapper"}    ← drift indicator
celery_task_runtime_seconds                    ← tasks getting slower?
gateway_sync_lag_minutes                       ← data staleness
api_request_duration_seconds                   ← latency
pod_restarts_total                             ← stability
db_connection_pool_available                   ← DB pressure
```

### Alerting Options
```
Current:  Email (slow, manual)
Better:
  Slack  → non-critical alerts (queue depth warning)
  PagerDuty/OpsGenie → critical (service down, queue > 10,000)
  SMS    → if truly critical and unacknowledged
```

### Flower Setup (Fix Your Partial Integration)
```bash
# Deploy as separate pod
celery -A your_app flower --port=5555 --broker=redis://redis:6379/0

# Add Prometheus metrics export
pip install flower  # has /metrics endpoint built in
# Prometheus scrapes flower:5555/metrics
# Alert: queue_depth > 1000 for > 5 mins → page on-call
```

---

## 6. Testing — BDD, TDD, Your Stack

### TDD
Write test first → RED → write code → GREEN → refactor → REFACTOR
Ensures testability by design, catches regressions immediately.

### BDD (Behaviour-Driven Development)
Tests in business language using Gherkin syntax:
```gherkin
Scenario: Gateway goes offline
  Given gateway "GW-001" was online
  When no heartbeat received for 20 minutes
  Then dashboard shows "GW-001" as Offline
  And alert email sent to operations team
```
Tools: `pytest-bdd`, `behave`

### Your pytest Hierarchy
```
Unit tests:       isolated functions, mock everything, fast, many
Functional tests: full request cycle, real test DB, mock external APIs
Integration tests: real external calls, slow, run less frequently
```

### CI Quality Gates Explained
```
Coverity   → static analysis: buffer overflows, null pointers, security issues
BlackDuck  → open source: license compliance (GPL risk), known CVEs in deps
SonarQube  → quality gate: coverage %, duplication, complexity, known patterns
All three must pass → block merge if any fail
```

---

## 7. Security Layers
```
Network:     Infoblox DNS + Azure Firewall → device-level filter
Gateway:     Kong → JWT validation, rate limiting, IP allowlist
Auth:        MSAL (UI users) + API Key/Bearer (services)
App:         Django middleware → tenant isolation, RBAC
Data:        Azure AD group membership → data access control
Secrets:     K8s Secrets + Azure Key Vault → no hardcoded credentials
Transport:   TLS at Kong, mTLS internally (zero-trust option)
Code:        Coverity + BlackDuck → vulnerability + license scanning
```

---

## 8. Component Alternatives at Different Scales

| Component | Small Scale | Your Scale (Honeywell) | Hyper Scale |
|-----------|------------|----------------------|-------------|
| Deployment | docker-compose on VM | AKS | Multi-region AKS/EKS |
| Queue | Celery + Redis | Celery + Redis | Kafka |
| Sync | Simple cron | Staggered cron + events | Full event-driven CDC |
| DB | Single Postgres | Postgres + read replicas | Cassandra / DynamoDB |
| Metrics DB | Postgres | TimescaleDB | InfluxDB |
| Monitoring | ELK only | ELK + Prometheus + Grafana | + Datadog / New Relic |
| API Gateway | Nginx | Kong | Kong + Azure APIM |
| Auth | Simple JWT | MSAL + Kong | MSAL + Kong + OPA (policies) |
| Alerts | Email | Email + Slack | PagerDuty + Slack |
| Cost | ~$200/month | $3,000-10,000/month | $50,000+/month |

---

## 9. Why Each Component Was Chosen

| Component | Why | Alternative |
|-----------|-----|-------------|
| Celery + Redis | Python-native, Django ecosystem, team familiarity | Kafka (complex, more power) |
| ELK | Standard, Kibana UI, team knows it | Loki+Grafana (lighter), Datadog (expensive) |
| AKS | Azure ecosystem, managed K8s, Honeywell is Microsoft shop | EKS (AWS), GKE (GCP) |
| Kong | Enterprise gateway, plugin ecosystem, self-hosted control | Azure APIM, AWS API GW |
| MSAL | Honeywell is on Azure AD, Microsoft stack | Okta, Auth0 |
| IoT Hub | Azure-native, managed device registry | AWS IoT Core, MQTT broker |
| Event Hub | Kafka-compatible, managed, Azure-native | Self-managed Kafka |
| PostgreSQL | Relational, ACID, good default, team knows it | MySQL, Cassandra (write-heavy) |
| Coverity | Industry standard for C/C++ security | Semgrep (open source) |
| BlackDuck | Enterprise license compliance | FOSSA, Snyk |
| JFrog | Private registry, enterprise security scanning | GitHub Container Registry |
| Octopus | Multi-tenant CD, environment promotion | ArgoCD (K8s-native), Flux |
| Flower | Celery-specific, visual queue monitoring | Prometheus + celery-exporter |

---

## 10. Interview Articulation — Honeywell Experience

> *"At Honeywell I built the portfolio intelligence platform for FOC — a multi-tenant dashboard showing gateway/site/customer/asset health across 38,000+ buildings. The system ingests IoT telemetry via Azure IoT Hub, syncs product mappings from external APIs every 5 hours via Celery Beat, and exposes data through a Django REST API with Kong as the API gateway. We encountered Celery queue drift — likely caused by burst task creation when the product mapper created thousands of subtasks simultaneously. The fix was a combination of staggered bucket processing, HPA on queue depth metrics, and properly configuring worker_max_tasks_per_child to prevent memory leak OOM kills. For observability we used ELK for logs and partially integrated Flower for Celery monitoring. Improving this with Prometheus + Grafana + PagerDuty would give us proper metric-based alerting before queue drift becomes a user-visible problem."*

---

## 11. Real-Time Data — Improvement Plan

### Why Current System Is Slow (Root Cause)
```
Current: PULL model (polling on a schedule)
  → system is blind between polls
  → data age = time since last poll (up to 5 hours for product mapper)
  → work done = full sweep of ALL gateways regardless of what changed

Better: PUSH model (react to changes)
  → system learns the moment something changes
  → work done = proportional to actual change rate
```

### Options (Best to Pragmatic)

**Option A — Webhooks from product source (ideal, if available)**
```
Product source → on any change → POST to your endpoint
→ process ONLY that gateway → seconds of latency
Zero polling, minimum work
```

**Option B — CDC on product source DB (if DB access available)**
```
Debezium reads product source WAL → Kafka/Event Hub
→ your consumer processes only changed rows
→ eliminates full sweep entirely
```

**Option C — Differential polling (pragmatic, available NOW)**
```python
# If product APIs support modified_after filter:
changed_gateways = product_api.get_gateways(modified_after=last_sync)
# Process ONLY changed gateways — could be 3 instead of 200K
# Run every 5 mins → latency drops from 5hrs to 5 mins
# No architectural change needed
```
**Ask: does your product source API support `modified_after` or `last_updated` filter?**
If yes → this alone solves most of the problem.

**Option D — Your proposed approach (staggered + scaled)**
```
Divide gateways into buckets, stagger processing:
Bucket 1 (50K gateways) → t=0
Bucket 2 (50K gateways) → t=7.5min
Bucket 3 (50K gateways) → t=15min
Bucket 4 (50K gateways) → t=22.5min
→ smooth load, no burst, queue never drifts
→ max data age = 30 mins
```
Still pull model but load is spread. Your while-loop consumer → replace with Celery/Event Hub consumer group (handles sleep/retry/DLQ natively).

### Dashboard Real-Time Options

**Frontend polling (current assumed):**
```
setInterval → GET /api/gateways every 30s → constant server load
Even when nothing changed — every user's browser fires every 30s
```

**SSE — Server-Sent Events (recommended for FOC dashboard):**
```
Browser: EventSource('/api/gateway-stream')
Server: pushes updates when data changes, client just receives
HTTP-native, auto-reconnects, works with existing LB/Nginx
Use when: read-only dashboard (server → client only)
```

**WebSockets:**
```
Bidirectional — overkill for a read-only dashboard
Use when: clients also send data back in real-time (chat, collaborative editing)
```

**Redis Pub/Sub as internal notification layer:**
```
CJ1/CJ2 updates DB → publishes to Redis channel
SSE endpoint subscribed to Redis channel
Moment data changes → SSE pushes to ALL connected browsers
~1-2 second end-to-end latency
```

### Impact Quantification (State in Interview)
```
Before: 5hr data age, full sweep of all gateways
Your fix: 30min data age, decoupled publisher/consumer, scalable consumers
Ideal: 5min (differential polling) or seconds (webhooks/CDC)
```

---

## 12. Fault Tolerance — Complete Picture

### What You Have
```
Service Bus DLQ      → messages survive CJ1 failure, processed on recovery
Azure Managed Postgres → Azure handles failover, automated backups
AKS pod restarts     → K8s restarts crashed pods automatically
Read replica         → reads continue if primary is slow
Redis cache-aside    → if Redis down, requests fall through to DB
```

### Idempotency in CJ1 — Critical Answer
```
If CJ1 processes "gateway created", crashes mid-write:
Service Bus redelivers → CJ1 processes same message again
Without idempotency: duplicate insert → data corruption

Fix: UPSERT instead of INSERT
  INSERT INTO gateway (...) 
  ON CONFLICT (gateway_id) DO UPDATE SET ...
  
Same message processed twice = same result → idempotent
```
**This is the most likely failure mode question. Have this answer ready.**

### CJ2 Partial Failure — The Hard Case
```
Product mapper processes 29/30 products, crashes on product 30
→ DB has mixed data: some mappings fresh, some stale
→ User sees inconsistent counts on dashboard (some products show new
  data, others show 6-hour-old data) — silently wrong

Fix A: Staging table + atomic swap
  Process all products → write to staging table
  On completion → atomic swap (UPDATE production FROM staging)
  → dashboard sees either fully old OR fully new data, never mixed

Fix B: Per-product sync timestamp
  Track last_synced_at per product
  Dashboard shows "last synced: X min ago" per product
  Partial failure is visible, not silently wrong
```

### Circuit Breaker on Product APIs
```
Product source P2 is rate-limited and sometimes slow
Without circuit breaker: P2 hangs → entire 4-hour job hangs

Fix: circuit breaker around each product source call
  If P2 fails N times → skip this cycle → log + alert → continue
  other products unaffected
  P2 picked up again next cycle when recovered
```

### What Happens to Service Bus Messages if CJ1 Fails
```
Service Bus retains messages (configurable TTL, default 14 days)
Failed/unprocessed messages → DLQ after max delivery count exceeded
On CJ1 recovery → processes from DLQ
Zero message loss if recovered within TTL window
```

### Redis Failure Handling
```
Cache-aside pattern: request → Redis miss → DB → repopulate
If Redis completely down → all requests hit DB directly
Read replica absorbs this if configured correctly
Should be explicitly tested: "Redis failover test"
```

---

## 13. Kong vs Nginx Ingress — The Two Routing Layers

```
Internet
    ↓
Kong (OUTSIDE AKS — outer gateway)
    ↓
AKS Cluster
    ↓
Nginx Ingress Controller (INSIDE AKS — inner router)
    ↓
Kubernetes Service (pod-level load balancer)
    ↓
Your Pods
```

### Kong — Outer Layer
```
Job: SSL/TLS termination, JWT validation, rate limiting,
     reverse proxy, route to correct AKS service/cluster
Load balancing: upstream round-robin to K8s service endpoint
Knows about: services/clusters, NOT individual pods
```

### Nginx Ingress — Inner Layer
```yaml
# What your YAML looks like:
spec:
  rules:
  - host: foc-service.internal
    http:
      paths:
      - path: /api → foc-backend-service:8000
      - path: /    → foc-frontend-service:80
```
Job: path/host-based routing to correct K8s Service inside cluster
Does NOT balance across pods — passes to K8s Service which does that
Cost efficiency: ONE Azure LB → Nginx → infinite routing rules (vs one LB per service)

### Kubernetes Service — Pod-Level Balancer
```
Maintains list of all healthy pod IPs via endpoint slices
kube-proxy (iptables) does round-robin across pods
Automatically removes pods that fail health checks
When HPA adds a new pod → pod passes readiness → auto-added to rotation
```

### The Three Layers — Different Concerns
```
Kong          → "Which cluster/service? Is this request allowed?"
Nginx Ingress → "Which K8s Service inside this cluster?"
K8s Service   → "Which specific running pod?"
```

### Interview One-liner
> "Kong handles SSL, auth, rate limiting, and routes to the correct AKS service. Inside AKS, Nginx Ingress routes by path/host to the correct Kubernetes Service. The K8s Service itself does pod-level load balancing via kube-proxy round-robin. They're complementary — Kong is our security boundary, Nginx is our internal traffic director, K8s Service is our pod distributor."

---

## 14. Scaling Strategy — Complete

```
High API traffic:      HPA scales web pods (CPU/memory threshold)
Product mapper lag:    Scale consumer pods based on Service Bus queue depth
DB read pressure:      Read replica (you have this)
DB write pressure:     PgBouncer connection pooling first, sharding if extreme
Service Bus throughput: Increase partition count if message volume grows
```

**Stateless services = critical for scaling:**
```
All FOC Service pods must be stateless — no local state
All state in Postgres or Redis
→ any pod can handle any request → HPA can freely add/remove pods
```

**Cache pre-warming after sync cycle:**
```
After CJ2 completes → proactively push fresh data to Redis
Eliminates cold-start cache miss storm when 5hr refresh happens
Every user gets cached fresh data immediately
```

---

## 15. Interview Q&A — Quick One-Liners

```
"Why Redis?"
→ Cache-aside for summary API responses + Celery broker. Hit rate
  is high because dashboard filters are limited — same queries repeat
  frequently across users viewing the same region/product/customer.

"Why Postgres over NoSQL?"
→ Data is relational — gateways to sites to customers to products.
  ACID guarantees matter — can't show inconsistent mapping counts.

"Why AKS over plain VMs?"
→ Auto-scaling (HPA), self-healing (pod restarts), rolling deployments
  (zero downtime), namespace isolation per tenant-environment.

"What's your biggest risk right now?"
→ Data consistency during CJ2 partial failures — a gateway's mapping
  might be half-updated if the job fails mid-run. Fix (staging table +
  atomic swap) is planned but not yet implemented.

"Why cron instead of event-driven for product mapper?"
→ Product sources don't expose webhooks or change streams — we're
  forced to poll. If they did, we'd switch to webhook/CDC immediately.
  Our improvement reduces latency from 5hrs to 30mins by decoupling
  publishing from consuming and scaling consumers independently.

"How do you handle the rate-limited product source?"
→ Retry with exponential backoff, circuit breaker to skip failing
  product and continue others, alerting if too many retries, DLQ
  for messages that fail after max retries.
```

---

## 16. Schema Design Points

```
Strengths:
  SGP (Site-Gateway-Product) table → flexible many-to-many without duplication
  Denormalization done deliberately → reduces atomic write complexity
  P-CGR table → multi-dimensional filtering without joins on every query
  history preserved → product DBs delete entries, you keep them

Risks:
  200K gateways × 30 products = up to 6M rows in SGP mapping table
  COUNT queries without proper indexes → slow at this scale
  Indexes needed on: gateway_id, product_id, customer_id, region
  Trend/history queries → time-based partitioning on gateway table
                           would improve range query performance

"What queries is this schema optimized for?"
→ Aggregated summary counts by region/product/customer (the main dashboard view)
→ Fast lookup: given a gateway_id, find all products/sites/customers
→ Not optimized for: ad-hoc cross-dimensional analytics
   (for that, consider a separate analytics/OLAP layer — dbt + BigQuery/Redshift)
```
