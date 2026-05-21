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
