# 🌐 NETWORKING
> **Status:** 🔴 Not started  
> **Target:** Solid understanding of how the web works — from DNS to TLS to HTTP internals

---

## Topic Checklist

### Fundamentals
- [ ] OSI model — what each layer does (don't just memorize names, know what breaks at each layer)
- [ ] TCP vs UDP — when each is appropriate, 3-way handshake, flow control
- [ ] IP addressing, subnetting, CIDR basics
- [ ] DNS — resolution process, TTL, types of records (A, CNAME, MX, TXT)
- [ ] HTTP/1.1 vs HTTP/2 vs HTTP/3 — what changed and why
- [ ] HTTPS / TLS — handshake, certificates, what SSL termination means
- [ ] Websockets — how they differ from HTTP, use cases (real-time)

### Web Infrastructure
- [ ] Load balancers — L4 vs L7, sticky sessions, health checks
- [ ] CDN — how it works, cache-control headers, origin vs edge
- [ ] Reverse proxy vs forward proxy
- [ ] NAT and how it works
- [ ] Firewalls, VPNs (conceptual level)

### Application Layer
- [ ] REST — statelessness, idempotency, status codes
- [ ] HTTP methods — GET/POST/PUT/PATCH/DELETE semantics
- [ ] Cookies, sessions, JWT — how auth works over HTTP
- [ ] CORS — why it exists, how it works
- [ ] gRPC — why it's faster, when to use over REST

### Performance & Reliability
- [ ] What happens when you type a URL and hit Enter (full walkthrough)
- [ ] Connection pooling
- [ ] Latency vs bandwidth — the difference
- [ ] Head-of-line blocking — HTTP/1.1 problem, how HTTP/2 solves it
- [ ] Long polling vs short polling vs SSE vs WebSockets

---

## Must-Know Deep Dives

### "What happens when you type google.com and press Enter?"
> This question is a proxy for knowing the full networking stack. Practice narrating it end to end:
> Browser cache → OS DNS cache → Recursive resolver → Root nameserver → TLD → Authoritative DNS → TCP handshake → TLS → HTTP GET → Response → Rendering

### TCP 3-Way Handshake
```
Client → SYN →             Server
Client ← SYN-ACK ←        Server
Client → ACK →             Server
[Connection established]
```

### HTTP/2 vs HTTP/1.1
| Feature | HTTP/1.1 | HTTP/2 |
|---------|----------|--------|
| Multiplexing | ❌ (one req/conn) | ✅ |
| Header compression | ❌ | ✅ (HPACK) |
| Server push | ❌ | ✅ |
| Binary protocol | ❌ (text) | ✅ |

---

## Notes
> Add your own observations here as you study

-

---

## Authentication Flows

### The Big Picture

```
Network layer  → DNS + Firewall  → Are you on the right network/device?
Identity layer → OIDC + MSAL     → Who are you? (client side)
Authz layer    → OAuth + JWT     → What are you allowed to do? (server side)
```

All three must pass. Valid device + wrong credentials = blocked at identity. Valid credentials + wrong device = blocked at network.

---

### OAuth 2.0 vs OpenID Connect (OIDC)

| | OAuth 2.0 | OpenID Connect (OIDC) |
|---|---|---|
| Answers | "Are you authorised?" | "Who are you?" |
| Produces | Access token | ID token |
| Lives | Server validates on every API call | Client uses once after login |
| Purpose | Authorization | Identity/authentication |

> OIDC = identity layer built ON TOP of OAuth 2.0.
> MSAL = library that orchestrates both.

---

### The Three Tokens

| Token | Purpose | Lifetime | Where |
|-------|---------|----------|-------|
| **Access token** | Proves authorisation to API. Sent in every request. | 1 hour | `Authorization: Bearer <token>` header |
| **ID token** | Contains user identity (email, name, groups). Who you are. | 1 hour | Decoded by client, not sent to APIs |
| **Refresh token** | Gets new access + ID tokens silently. No re-login needed. | 24hrs-90days | Browser storage, never sent to your API |

Access token ≠ ID token. Common confusion. Access = authorisation. ID = identity.

---

### Exact Sequence — Honeywell MSAL Flow

```
1. Hit foc-utility.honeywell.com
2. MSAL checks: valid token in browser? NO
3. Redirect to login.microsoftonline.com (OIDC happens here)
4. User logs in → Microsoft issues: access token + ID token + refresh token
5. MSAL stores tokens in browser memory
6. User lands on app ← MSAL + OIDC complete, all client side
      ↓
7. First API call:
   MSAL auto-attaches: Authorization: Bearer <access_token>
8. Backend validates JWT:
   - Decode header + payload (base64, anyone can decode)
   - Verify signature using Microsoft's public keys
   - Check: issuer, audience, expiry, email in Azure AD
   - Extract groups → check permissions → allow/deny
   ← OAuth happens here, server side
```

---

### Silent Token Refresh

```
Access token expires (1hr)
      ↓
MSAL intercepts 401 response
      ↓
Has valid refresh token? YES
      ↓
Silent POST to Microsoft: { refresh_token, grant_type: refresh_token }
      ↓
New access token issued → retry original API call
User notices nothing

Refresh token also expired?
      ↓
Redirect to login page — user must authenticate again
```

---

### JWT Structure

```
Header.Payload.Signature
  ↓         ↓         ↓
base64    base64    cryptographic
(algo)   (claims)   signature

Claims in payload:
  iss = issuer (Microsoft)
  aud = audience (your app client ID)
  exp = expiry timestamp
  email/upn = user identity
  groups = Azure AD group memberships
```

Anyone can decode a JWT — it's just base64. Security comes from the signature — only Microsoft can produce a valid one. Your backend verifies using Microsoft's published public keys.

---

### The Four OAuth Flows

| Flow | Use case |
|------|---------|
| **Authorization Code + PKCE** | Web apps, SPAs (what Honeywell uses) |
| **Client Credentials** | Service-to-service, background jobs, no user involved |
| **Device Code** | IoT devices, CLIs, no browser available |
| **Authorization Code (with secret)** | Web apps with secure backend (older pattern) |

---

### Interview One-liner

> *"We use OAuth 2.0 Authorization Code flow with PKCE via MSAL. MSAL handles OIDC on the client — establishes who you are, stores tokens, silently refreshes every hour. The backend validates the JWT signature on every API call using Microsoft's public keys, extracts claims, and checks Azure AD group membership for authorization. Network layer (DNS + firewall) provides the first guard — only corporate devices reach the app."*

---

## API Authentication Mechanisms

### Mental Model First

```
Bearer Token = HOW you send credentials (transport mechanism)
OAuth 2.0    = FRAMEWORK that issues those tokens
JWT          = FORMAT of the token

OAuth produces Bearer tokens.
Bearer tokens carry JWTs.
These are not alternatives — they work together.
```

---

### 1. Basic Auth
```
Authorization: Basic base64("username:password")
```
- Sends credentials on EVERY request
- base64 is NOT encryption — anyone can decode it
- Must always use HTTPS
- Use only for: internal tools, scripts, legacy systems
- Never for: public APIs, anything user-facing

---

### 2. Bearer Token (JWT)
```
Authorization: Bearer eyJhbGc...
```
- Token proves identity — "whoever bears this is allowed in"
- Two types:
  - **Opaque** = random string, server does DB lookup
  - **JWT** = self-contained, server validates signature (no DB lookup)
- JWT is the modern standard — stateless, scalable
- What Honeywell/MSAL uses

**JWT structure:**
```
Header.Payload.Signature
base64  base64  cryptographic
(algo) (claims)  signature
```
Anyone can decode — security comes from signature only Microsoft can produce.

---

### 3. API Key
```
X-API-Key: sk_live_abc123xyz
```
- Static secret issued to an application (not a user)
- Identifies an app, not a person
- Use for: third-party developer access (Stripe, Google Maps pattern)
- Risk: static, no expiry — leaked = valid forever until rotated

---

### 4. OAuth 2.0 Client Credentials
```
POST /oauth/token
{ grant_type: "client_credentials", client_id, client_secret }
→ returns access token (Bearer JWT)
```
- Service authenticates as itself — no user involved
- Use for: service-to-service, Celery workers, background jobs
- Machine equivalent of Authorization Code flow

---

### 5. mTLS (Mutual TLS)
- Both client AND server present certificates
- Normal HTTPS: server proves identity to client only
- mTLS: both sides prove identity
- Use for: high security microservice mesh, zero-trust architecture
- Overkill for user-facing APIs

---

### 6. Session Cookie
```
Set-Cookie: session_id=abc123; HttpOnly; Secure
```
- Server stores session state, gives client a cookie
- Every request sends cookie automatically
- vs JWT: session = server stores state (DB/Redis lookup per request)
          JWT = stateless (no DB lookup, verify signature only)
- JWT more scalable. Session easier to invalidate.

---

### Decision Framework

```
Human user (web/mobile)    → OAuth 2.0 Auth Code + PKCE → Bearer JWT
Service/machine            → OAuth 2.0 Client Credentials → Bearer JWT
Third-party developer      → API Key
Internal tool/script       → Basic Auth (HTTPS only) or API Key
High-security service mesh → mTLS
Legacy/simple system       → Session Cookie
```

---

## Forward Proxy vs Reverse Proxy

```
Forward Proxy  = sits in front of CLIENT  → hides client from server
Reverse Proxy  = sits in front of SERVER  → hides server from client
```

### Forward Proxy
```
Client → Forward Proxy → Internet → Server
Server sees: proxy IP, not client IP
```
- Configured on CLIENT side
- Use: corporate filtering (Honeywell monitors outbound traffic), VPNs, anonymity
- Your Honeywell laptop → corporate proxy → internet

### Reverse Proxy
```
Client → Reverse Proxy → Backend Servers
Client sees: proxy IP/domain only, never backend IPs
```
- Configured on SERVER side, client doesn't know it exists
- Every large website uses one — Google, Netflix, all of them
- Use: load balancing, TLS termination, caching, routing, security
- Examples: Nginx, Kong, AWS ALB, Traefik

**The family tree:**
```
Reverse Proxy      → forwards requests, hides backends
Load Balancer      → reverse proxy + traffic distribution
API Gateway        → reverse proxy + auth + rate limiting + routing + plugins
CDN                → reverse proxy + geographically distributed caching
K8s Ingress        → reverse proxy inside a cluster
```
All of these ARE reverse proxies with extra capabilities.

### DNS vs Proxy — When Each Comes In
```
1. DNS resolves domain → IP (happens BEFORE connection)
2. TCP connection established to that IP (reverse proxy's IP)
3. Reverse proxy routes to correct backend
```
DNS = phonebook (domain → IP). Reverse proxy = receptionist (routes to correct server).

---

## HTTP Versions — Persistent Connections, Multiplexing, QUIC

### The Connection Problem

Every HTTP request needs a TCP connection. TCP handshake = 1 RTT (round trip). At 150ms to a US server = 150ms wasted before data flows.

### HTTP/1.0 — New connection per request
```
Open TCP → GET /index.html → response → CLOSE
Open TCP → GET /style.css  → response → CLOSE
Open TCP → GET /logo.png   → response → CLOSE
30 resources = 30 TCP handshakes = extremely slow
```

### HTTP/1.1 — Persistent Connections (Keep-Alive)
```
Open TCP (once)
  → GET /index.html → response
  → GET /style.css  → response
  → GET /logo.png   → response
Close TCP
```
One handshake, many requests. **Default in HTTP/1.1.**
REST APIs use this — multiple API calls reuse the same TCP connection transparently.

**Problem: Head-of-line blocking**
Requests are sequential on one connection. Slow first request blocks everything behind it.
```
GET /slow-api  (200ms) → must complete before...
GET /fast-api  (10ms)  → ...this can start
Total: 210ms sequential
```
Browser workaround: open 6 parallel TCP connections per domain.

### HTTP/2 — Multiplexing
One TCP connection, multiple **streams** simultaneously. Responses arrive in any order.
```
Stream 1: GET /api/users    → response (200ms)
Stream 2: GET /api/orders   → response (50ms)  ← arrives first
Stream 3: GET /api/products → response (100ms)
Total: 200ms (limited by slowest, not sum)
```

**Other HTTP/2 features:**
- **Header compression (HPACK)** — don't repeat same headers every request
- **Binary protocol** — more efficient than text
- **Server push** — server sends resources before client asks

**Remaining problem: TCP head-of-line blocking**
One lost TCP packet blocks ALL streams (TCP guarantees ordered delivery). HTTP/2 over bad networks can be worse than HTTP/1.1.

### HTTP/3 — QUIC (UDP-based)
Replaces TCP with **QUIC** — built on UDP, implements streams at transport layer.

```
Lost packet in Stream 1 → only Stream 1 waits
Stream 2, 3 unaffected ✅
```

**Other QUIC features:**
- **1 RTT** first connection (vs 2-3 RTT for HTTP/2 + TLS)
- **0-RTT reconnection** — client sends data immediately on reconnect
- **Connection migration** — connection survives network switch (WiFi → 4G)
  Connection ID-based, not IP+port based. Perfect for mobile.

### Comparison Table
| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Transport | TCP | TCP | QUIC (UDP) |
| Persistent connections | ✅ | ✅ | ✅ |
| Multiplexing | ❌ | ✅ | ✅ |
| HTTP HOL blocking | ❌ | ✅ fixed | ✅ fixed |
| TCP HOL blocking | ❌ | ❌ | ✅ fixed |
| Header compression | ❌ | ✅ HPACK | ✅ QPACK |
| 0-RTT reconnect | ❌ | ❌ | ✅ |
| Connection migration | ❌ | ❌ | ✅ |

### WebSockets vs HTTP Persistent Connections
```
HTTP persistent:  client always initiates, request→response pattern
WebSocket:        bidirectional, either side sends anytime

WebSocket upgrade:
Client: GET /chat + Upgrade: websocket
Server: 101 Switching Protocols → now full duplex
```
Use WebSockets for: chat, live notifications, real-time dashboards, collaborative editing.

---

## TLS / SSL

### SSL vs TLS
SSL is deprecated (broken). TLS replaced it. TLS 1.2 widely used, TLS 1.3 is current standard.
"SSL certificate" = "TLS certificate" in casual usage. Same thing.

### What TLS Solves
```
Without TLS: POST /login { password: "secret" } → readable by anyone on network
TLS solves:
  1. Encryption    → nobody can read data in transit
  2. Authentication → you're talking to the REAL server, not an impostor
```

### Symmetric vs Asymmetric Encryption
```
Symmetric:   same key encrypts + decrypts. Fast. Problem: how to share the key?
Asymmetric:  public key encrypts, private key decrypts. Slow. Solves key sharing.

TLS uses both:
  Asymmetric → securely exchange a symmetric session key
  Symmetric  → encrypt actual data (fast)
```

### Certificate + Certificate Authority (CA)
A certificate says: "this public key belongs to honeywell.com, verified by DigiCert"

Contains: domain name, server's public key, CA signature, expiry date.

Browser ships with list of trusted CAs. CA verifies domain ownership and signs cert.
```
Root CA (DigiCert) → signs Intermediate CA → signs honeywell.com cert
Chain of trust: browser trusts Root CA → trusts everything it signed
```
Root CA private key kept offline in vault. Intermediate CA does day-to-day signing.

### TLS 1.2 Handshake (2 RTT)
```
Client → ClientHello (TLS version, cipher suites, client random)
Server → ServerHello (chosen cipher, server random) + Certificate
Client → verifies cert (trusted CA? correct domain? not expired?)
Client → PreMasterSecret (encrypted with server's public key)
Both   → derive session key = f(client random + server random + premaster secret)
Both   → Finished (encrypted) — handshake complete
Data flows encrypted with symmetric session key
```
Session key is NEVER transmitted — derived independently by both sides.

### TLS 1.3 Improvements
- **1 RTT** (vs 2 RTT) — client sends key share upfront
- **0-RTT resumption** — reconnect with zero round trips
- **Forward secrecy mandatory** — ephemeral keys per session
  - If private key later stolen → past sessions still safe
- **Removed weak ciphers** — smaller attack surface

### TLS Termination (Your Honeywell Architecture)
```
Client → HTTPS (encrypted) → Kong → HTTP (unencrypted, internal) → Pod
```
Kong handles TLS — app pods do business logic only.
Benefits: centralised cert management, L7 routing possible, no crypto overhead on pods.

### TLS One-liner
> *"TLS = asymmetric crypto to securely exchange a symmetric session key, then symmetric crypto for data. Certificate proves server identity via CA chain of trust. TLS 1.3 = 1 RTT, forward secrecy, 0-RTT reconnect. Terminates at Kong in our architecture."*

---

## API Gateway Landscape

| Gateway | Type | Best for |
|---------|------|---------|
| Kong | Self-hosted, open source | Enterprise control, K8s, plugin ecosystem |
| AWS API Gateway | Managed (AWS) | AWS ecosystem, Lambda integration, zero ops |
| Azure APIM | Managed (Azure) | Azure ecosystem, developer portal, deep AD integration |
| Apigee (GCP) | Managed (GCP) | Enterprise analytics, API monetization |
| Traefik | Self-hosted, K8s-native | K8s auto-discovery, simpler than Kong |
| Nginx | Self-hosted | Lightweight reverse proxy, basic routing |
| Envoy | Data plane | Service mesh (Istio), east-west traffic |

### North-South vs East-West Traffic
```
North-South = external clients → your services
  → API Gateway handles this (Kong, APIM, AWS API GW)

East-West = service to service (internal)
  → Service mesh handles this (Istio + Envoy, Linkerd)
```

### Honeywell Stack
```
Internet → Azure Firewall → Kong (north-south gateway)
        → AKS → Nginx Ingress (cluster routing) → Pods
```
