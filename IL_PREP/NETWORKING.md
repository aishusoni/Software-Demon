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
