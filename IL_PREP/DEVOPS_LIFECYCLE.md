# 🚀 DEVOPS LIFECYCLE — Honeywell Pipeline
> Full CI/CD pipeline walkthrough — from code commit to production pod
> Use this to articulate your engineering experience in interviews

---

## The Full Pipeline

```
Developer → GitHub → CI (GitHub Actions) → Docker Image → JFrog → CD (Octopus) → AKS → User
```

---

## 1. Git + GitHub

- **Git** = local version control tool
- **GitHub** = remote store + collaboration layer
- **Branch strategy:** main/master is protected. Developers work on feature branches. PRs trigger CI.
- Every PR → GitHub Actions pipeline fires automatically

---

## 2. CI — GitHub Actions

Defined in `.github/workflows/*.yml`. GitHub spins up a fresh VM and runs the pipeline on every PR.

**Pipeline stages:**
```
Trigger: PR raised or push to branch
      ↓
Checkout code
      ↓
Install dependencies
      ↓
Unit tests           ← catches logic bugs
Integration tests    ← catches component interaction bugs
Linting              ← code style, formatting
      ↓
Third-party checks (parallel):
  Coverity    ← static analysis, security vulnerabilities (buffer overflows, null pointers)
  BlackDuck   ← open source license compliance (GPL in commercial = legal risk), CVE scanning
  SonarQube   ← code quality gate (coverage %, duplication, complexity)
      ↓
All pass? → Build Docker image tagged with commit hash
      ↓
Push image to JFrog Artifactory
```

**The three tools:**

| Tool | What it checks | Why it matters |
|------|---------------|----------------|
| Coverity | Security vulnerabilities in code (static analysis) | Industrial software has strict security requirements |
| BlackDuck | Open source license compliance + known CVEs in dependencies | GPL code in commercial product = legal problem |
| SonarQube | Code quality gates — coverage %, duplication, complexity | Enforces team standards, prevents tech debt |

---

## 3. Docker Image

**What it is:** Snapshot of your entire application — code, runtime, dependencies, OS libraries — packaged into one portable artifact. Runs identically anywhere.

```dockerfile
FROM python:3.11
COPY . /app
RUN pip install -r requirements.txt
CMD ["python", "manage.py", "runserver"]
```

**Image tagging with commit hash:**
```
your-app:a3f9c2b    ← tagged with git commit hash
```

Why: Full traceability. Bug in prod → know exact commit. Roll back to `your-app:prev-hash` instantly.

---

## 4. JFrog Artifactory

**What it is:** Private artifact repository for your organisation. Like Docker Hub but private + secure.

- Stores Docker images, JAR files, npm packages
- Images can't go to public Docker Hub (proprietary code)
- Also does vulnerability scanning on images before deployment
- Octopus pulls from here

---

## 5. Octopus Deploy — CD

**What it does:**
- Connects to JFrog, lists available images with commit hashes + metadata
- You select: image, tenant, environment, namespace
- Generates Kubernetes manifests and applies to AKS
- Tracks deployment history — who deployed what, when, previous version

**Environments in pipeline:**
```
dev → staging → prod
```

Each is a separate K8s namespace or cluster. Octopus manages promotion between them.

---

## 6. Kubernetes + AKS — Core Concepts

**AKS** = Azure Kubernetes Service. Azure manages the K8s control plane. You manage workloads.

### Hierarchy (bottom up)

**Container** → single running instance of a Docker image

**Pod** → wraps one or more containers. Smallest deployable unit. Gets its own cluster IP. Ephemeral — can be killed/replaced anytime.

**Node** → a VM running multiple pods
```
Node (8 vCPU, 32GB RAM)
├── Pod A (Django app)
├── Pod B (Django app)
└── Pod C (Celery worker)
```

**Deployment** → declares desired state: "always run 3 replicas of this pod, use this image, rolling-update when I change the image"

**Service** → stable IP/DNS name pointing to a group of pods. Load balances across them. Pods come and go — Service address stays constant.

**Namespace** → logical partition inside cluster. Isolates resources between teams/environments/tenants.

**HPA (Horizontal Pod Autoscaler)** → auto-scales pod count based on CPU/memory
```
CPU > 70% → scale up pods
CPU < 30% → scale down pods
Cooldown:  3-5 minutes between scale events
```

### The YAML file

Everything in K8s is defined as YAML:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: foc-utility
  namespace: tenant-a-prod
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: foc-utility
        image: jfrog.io/foc-utility:a3f9c2b
        ports:
        - containerPort: 8000
        resources:
          requests:             # guaranteed minimum
            memory: "512Mi"
            cpu: "250m"
          limits:               # hard maximum (OOM kill if exceeded)
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:          # is pod alive?
          httpGet:
            path: /health
            port: 8000
          periodSeconds: 10
        readinessProbe:         # is pod ready for traffic?
          httpGet:
            path: /ready
            port: 8000
          initialDelaySeconds: 30
```

### Health Checks

**Liveness probe:** Is pod alive? Fail 3 times → pod killed, new one created.

**Readiness probe:** Is pod ready for traffic? Fail → removed from load balancer (not killed). This enables zero-downtime rolling deployments — new pods only receive traffic once ready.

### Rolling Deployments (zero downtime)
```
Deploy new image:
  Create 1 new pod (new image) → wait for readiness → 
  Remove 1 old pod → 
  Repeat until all pods updated
```
Service always points to at least some healthy pods throughout.

### What happens when a node dies
K8s control plane detects node is gone. All pods rescheduled onto other nodes automatically. This is why you run 3+ replicas — one node dying doesn't take down the service.

---

## 7. Tenants + Environments

**Tenant** = a customer organisation or business unit
```
Tenant A: Honeywell Aerospace division
Tenant B: Honeywell Building Technologies - Singapore
Tenant C: External enterprise customer
```

**Environment** = stage in deployment pipeline
```
dev → staging → prod
```

**How they map to Kubernetes namespaces:**

**Pattern 1 — Namespace per tenant per environment:**
```
Cluster
├── Namespace: tenant-a-dev
├── Namespace: tenant-a-prod
├── Namespace: tenant-b-dev
└── Namespace: tenant-b-prod
```

Octopus selects: image + tenant + environment → maps to correct namespace → deploys.

**Pattern 2 — Config-driven tenant isolation:**
Same namespace, tenant injected via ConfigMap/environment variable:
```yaml
env:
  - name: TENANT_ID
    value: "tenant-a"
  - name: DB_CONNECTION
    valueFrom:
      secretKeyRef:
        name: tenant-a-db-secret
        key: connection_string
```

App reads `TENANT_ID` at runtime, serves correct data/config.

---

## 8. ConfigMaps + Secrets

Never hardcode passwords or API keys in YAML/code.

**ConfigMap** — non-sensitive config (feature flags, URLs, settings)
**Secret** — sensitive data (DB passwords, API tokens, certificates) — stored encrypted

Both injected as environment variables into pods at runtime.

---

## 9. Nginx Ingress Controller

Nginx runs **inside** the K8s cluster as a pod. Handles ALL incoming HTTP/HTTPS traffic — both frontend and backend.

**Request path:**
```
User: foc-utility.honeywell.com
      ↓
DNS → Azure Load Balancer
      ↓
Nginx Ingress Controller (pod in AKS)
      ↓
Routes based on Ingress rules:
  /api/* → Django backend Service → backend pods
  /      → Frontend Service → frontend pods
      ↓
TLS termination (HTTPS outside, HTTP inside cluster)
```

**Ingress YAML:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: foc-utility-ingress
spec:
  rules:
  - host: foc-utility.honeywell.com
    http:
      paths:
      - path: /api
        backend:
          service:
            name: foc-utility-backend
            port: 8000
      - path: /
        backend:
          service:
            name: foc-utility-frontend
            port: 80
```

This is why users see `foc-utility.something.com` — Ingress maps domain → service.

---

## 10. Celery in AKS

Celery runs as **separate Deployments** in the same namespace:

```
Namespace: tenant-a-prod
├── Deployment: foc-utility-web          (Django, 3 replicas, scales with HPA)
├── Deployment: foc-utility-celery-worker (Celery workers, 2+ replicas, scales with HPA)
└── Deployment: foc-utility-celery-beat   (Beat scheduler, ALWAYS 1 replica)
```

**Why Beat must be 1 replica:** Multiple Beat instances each independently trigger the same scheduled task → 3× execution. Beat is a singleton.

Workers scale freely — more workers = more parallel task processing.

---

## Interview Articulation

> *"Our pipeline uses GitHub Actions for CI — unit tests, integration tests, plus Coverity for security static analysis, BlackDuck for license compliance, and SonarQube as a quality gate. On pass, we build a Docker image tagged with the commit hash and push to JFrog Artifactory. Octopus handles CD — it pulls the image and deploys to AKS, where we use namespace-level isolation per tenant-environment combination. Nginx Ingress Controller handles routing inside the cluster. Each deployment uses rolling updates with readiness probes for zero downtime. Celery workers run as separate deployments with HPA scaling, with Beat as a singleton to avoid duplicate task execution."*

That's a complete, senior-level answer to "walk me through your deployment pipeline."

---

## 11. Kong — API Gateway (Facade Layer)

**What a facade layer means:**
Single entry point that hides all internal complexity. Clients talk to ONE endpoint. Kong figures out where to route things internally.

**Kong vs Nginx — the distinction:**
```
Reverse Proxy  → forwards requests, hides backend servers
Load Balancer  → distributes traffic across servers
API Gateway    → auth + rate limiting + routing + logging + plugins
```
Every API gateway is a reverse proxy. Not every reverse proxy is an API gateway.

**Kong is built on Nginx + OpenResty (Lua scripting).**
But configured dynamically via API, not static nginx.conf files.

**What Kong does in Honeywell architecture:**
```
Client request
      ↓
Kong API Gateway
  ├── JWT validation      → reject invalid tokens before hitting AKS
  ├── Rate limiting       → per user/service limits
  ├── Request routing     → /foc/* → FOC cluster
  │                          /building/* → Building Intelligence cluster
  ├── Request transform   → add/strip headers
  ├── Central logging     → every request logged
  └── Analytics           → latency, error rates per service
      ↓
Correct AKS cluster → Nginx Ingress → Pods
```

**Two routing layers in Honeywell:**
```
Kong            → which cluster/service? (outer layer)
Nginx Ingress   → which service inside cluster? (inner layer)
K8s Service     → which pod inside service? (innermost)
```

**Why the facade pattern matters:**
- Security: backend IPs/ports never exposed to internet
- Flexibility: change backend architecture without clients knowing
- Single enforcement point: auth/rate-limit patch in Kong protects all services

**Full Honeywell request path:**
```
Honeywell device (corporate network)
      ↓
DNS resolution (internal only)
      ↓
Azure Firewall
      ↓
Kong API Gateway (validate JWT, rate limit, route)
      ↓
AKS Cluster → Nginx Ingress → K8s Service → Pod
      ↓
Application-level auth (groups, permissions)
      ↓
Business logic → Response
```

---

## 12. Docker — Local Development + Containerization

### Core Mental Model
```
Dockerfile  → recipe (instructions to build an image)
Image       → built artifact (read-only, layered)
Container   → running instance of an image
```

### Dockerfile — Backend (Python/Django)
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .           # copy this FIRST
RUN pip install -r requirements.txt  # cached if requirements unchanged
COPY . .                          # code changes don't invalidate pip cache
EXPOSE 8000
CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```
`0.0.0.0` not `127.0.0.1` — localhost inside container = unreachable from outside.
Layer caching: copy requirements before code — pip layer reused when only code changes.

### Dockerfile — Frontend (React, multi-stage)
```dockerfile
FROM node:20 AS build          # stage 1: heavy build environment
WORKDIR /app
COPY package*.json .
RUN npm install
COPY . .
RUN npm run build

FROM nginx:alpine               # stage 2: tiny runtime, just static files
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```
Multi-stage: final image carries only built output, not 1GB of node_modules.

### docker-compose.yml — Full Stack
```yaml
version: '3.8'
services:
  db:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: myapp
    volumes:
      - pgdata:/var/lib/postgresql/data    # data survives container restarts
    ports:
      - "5432:5432"

  backend:
    build: ./backend
    depends_on:
      - db
    environment:
      DATABASE_URL: postgresql://postgres:secret@db:5432/myapp
    ports:
      - "8000:8000"

  frontend:
    build: ./frontend
    ports:
      - "3000:80"

volumes:
  pgdata:
```

CRITICAL: Service names ARE hostnames inside Docker network.
Backend connects to DB via `db:5432`, NOT `localhost:5432`.

### .dockerignore
```
node_modules
.git
__pycache__
*.pyc
.env
```
Without this: builds include node_modules (~1GB), massively slow.

### Key Commands
```bash
docker-compose up --build       # build images + start everything
docker-compose up -d            # run in background
docker-compose down             # stop everything
docker-compose logs backend     # logs for one service
docker exec -it <id> bash       # shell into running container (debug)
docker system prune -a          # clean up unused images/containers
docker ps                       # list running containers
docker images                   # list all images
```

### Production Deployment Path
```
1. Local: docker-compose up --build  (verify it works)
2. Push code to GitHub
3. CI (GitHub Actions): run tests → build image → push to registry
4. Registry: Docker Hub / ghcr.io (free) vs JFrog (enterprise)
5. Deploy:
   Simple  → Render/Railway/Fly.io (connect repo, auto-deploy, free HTTPS)
   VM      → EC2/DigitalOcean, SSH, docker-compose up -d, Nginx + certbot SSL
   K8s     → image → registry → kubectl apply → Ingress → Service → Pod
6. Secrets: .env locally (gitignored), platform secret manager in prod
7. DNS: A record → VM IP, or CNAME → platform URL
8. SSL: Let's Encrypt (certbot) for VM, automatic on managed platforms
```

### AI-Assisted Coding Round Tips
- Prompt incrementally, not in one giant ask — review each piece before continuing
- Read every line before accepting — know why each line is there
- Let AI handle boilerplate (CRUD, Dockerfiles, component shells)
- Use your brain for edge cases, error handling, business logic
- Ask AI to generate test cases — shows engineering maturity
- Say prompts out loud if observed — interviewers evaluate how you direct the tool
- `/cost` or `/usage` in Claude Code to track token consumption
- `/compact` between major milestones to free context
- `/model sonnet` for most work, `/model opus` for complex architecture

---

## 13. Docker Compose — Deep Dive

### Volumes — Why They Exist
Containers are ephemeral — delete/recreate destroys data inside. Volumes persist data outside the container's lifecycle.
```yaml
volumes:
  - pgdata:/var/lib/postgresql/data   # named volume — Docker manages location
  - .:/app                            # bind mount — maps local folder (DEV ONLY, live reload)
```
Named volume = production pattern for DB data. Bind mount = dev-only convenience.

### Networking — Service Names ARE Hostnames
```
Inside `web` container, connect to DB via: db:5432
NOT localhost:5432 — Docker Compose creates internal DNS,
each service reachable by its service name
```
This trips up almost everyone the first time.

### `depends_on` Doesn't Wait for Readiness
```yaml
depends_on:
  - db    # only controls START ORDER, NOT readiness
```
Postgres container starts but isn't ready to accept connections yet → web container may fail to connect.

**Fix — healthcheck:**
```yaml
db:
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U postgres"]
    interval: 5s
    retries: 5
web:
  depends_on:
    db:
      condition: service_healthy   # NOW actually waits
```

### Lifecycle Commands
```bash
docker-compose build          # build images only
docker-compose up -d          # build + start, detached
docker-compose down           # stop + remove containers (volumes survive)
docker-compose down -v        # also removes volumes — DATA GONE
docker-compose logs -f web    # follow logs for one service
docker-compose exec web bash  # shell into running container
```

---

## 14. CDN vs WhiteNoise — Static File Serving

```
WhiteNoise: runs INSIDE your Django app, your Gunicorn process serves files
  Use when: small app, low traffic, simplicity > shaving milliseconds

CDN: separate globally distributed network, caches files near each user
  Use when: global users, high traffic, large assets, offload bandwidth

WhiteNoise: User → YOUR single server (every request)
CDN:        User → nearest edge (cached after first fetch) → origin only on MISS
```
For a small app: WhiteNoise is correct, proportionate. CDN now = solving a problem you don't have yet.

---

## 15. WSGI, ASGI, Gunicorn — Full Explanation

### Web Server vs Application Server (terminology precision)
```
"Web server"          = traditionally Nginx/Apache — serves static files,
                         handles raw connections efficiently
"Application server"  = Gunicorn/uWSGI — runs YOUR application code per request
```
Colloquially "web server" is used loosely for both. Precise term for Gunicorn: WSGI application server.

### WSGI — The Contract, Not a Program
WSGI = Web Server Gateway Interface. A SPECIFICATION — any WSGI-compliant server can run any WSGI-compliant Python framework.
```python
# What Django generates for you in wsgi.py:
application = get_wsgi_application()
# Conceptually: def application(environ, start_response): ...
```
Because it's a standard contract, the web server and framework don't need to know about each other beyond this interface.

### Gunicorn — An Actual Program Implementing WSGI
```
1. Listens on a port
2. Accepts TCP, parses raw HTTP → WSGI environ dict
3. Calls application(environ, start_response)
4. Takes Django's response → converts to raw HTTP bytes
5. Sends back to client
6. Manages WORKER PROCESSES for concurrency
```
```bash
gunicorn yourproject.wsgi:application --workers 4 --bind 0.0.0.0:8000
```
Why not `runserver`: dev-only, single-threaded by default behavior, not hardened, no worker process management/auto-restart.

### Full Request Flow
```
Browser → Nginx (TLS termination, static files, port 443)
        → Gunicorn (internal port, WSGI server, manages workers)
        → application(environ, start_response)  [WSGI CONTRACT]
        → Django (your actual code)
        → response flows back up
```
WSGI/ASGI = the rulebook. Gunicorn/Uvicorn = a program following the rulebook. Django = your app code, also following the rulebook.

### ASGI — The Async Successor
WSGI is fully synchronous — cannot handle WebSockets, long-lived connections, native async views.
```python
async def application(scope, receive, send): ...
# yourproject/asgi.py
application = get_asgi_application()
```
Servers: Uvicorn (most common), Daphne (built for Django Channels), Hypercorn.
```bash
gunicorn yourproject.asgi:application -k uvicorn.workers.UvicornWorker --workers 4
```
(Gunicorn's process management + Uvicorn's async handling underneath)

**When you need ASGI:** WebSockets, Django Channels, real-time push, SSE.
**When WSGI is sufficient:** standard CRUD apps, request-response only (e.g., reading list app).

### One-liner
> *"WSGI is the synchronous contract between Python web servers and frameworks — Gunicorn implements it as a real server, managing worker processes. ASGI is the async successor for WebSockets/long-lived connections, served by Uvicorn/Daphne. Standard CRUD apps only need WSGI + Gunicorn."*
