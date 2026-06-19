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
