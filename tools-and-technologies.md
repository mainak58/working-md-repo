# System Design Roadmap — Progress Checklist

> **Paste this into a GitHub Issue, not a repo file.** In issues, pull requests, and discussions the checkboxes are clickable and GitHub saves each tick for you. In a `.md` file inside a repo they render as read-only boxes — you'd have to edit `[ ]` to `[x]` by hand.
>
> GitHub also shows a progress bar (e.g. "12 of 95") next to the issue in your issue list, as long as the task list is in the **first** comment.

**How to use it:** new issue → title it `System Design Roadmap` → paste everything below → submit. Then just click boxes as you go.

Each topic has three boxes:
- **Learn** — read or watch until you can explain it out loud
- **Do** — write code or run the thing yourself
- **Say** — could you answer the question with no notes?

Don't tick a topic until all three are done. Reading alone doesn't count.

---

## Progress Tracker

| Phase | Topics | Done |
|---|---|---|
| 0 — Prerequisites | 5 | ` /5 ` |
| 1 — Traffic & Edge | 4 | ` /4 ` |
| 2 — Database | 3 | ` /3 ` |
| 3 — Scaling | 3 | ` /3 ` |
| 4 — Real-Time | 4 | ` /4 ` |
| 5 — Messaging | 3 | ` /3 ` |
| 6 — Deployment | 3 | ` /3 ` |
| 7 — Observability | 3 | ` /3 ` |
| 8 — Performance | 2 | ` /2 ` |
| 9 — Optional | 2 | ` /2 ` |
| **Total** | **32** | ` /32 ` |

**Started:** ______________ **Target finish:** ______________

---

## Phase 0 — Prerequisites

- [ ] **HTTP fundamentals** — methods, status codes, headers, cookies
- [ ] **Client / server / port** — what they actually are
- [ ] **Linux command line** — navigating, permissions, processes, logs
- [ ] **One backend language + one database** — enough to build CRUD
- [ ] **Git** — branches, merges, rebase, resolving conflicts

**Milestone project**
- [ ] Built a small REST API with a database behind it, running locally

---

## Phase 1 — Getting Traffic to Your Server

### 1. Nginx `Tool`
- [ ] Learn — reverse proxy, static files, TLS termination, config syntax
- [ ] Do — put Nginx in front of your API
- [ ] Explain — why do we need a reverse proxy at all?

### 2. Load Balancing `Concept`
- [ ] Learn — round-robin, least-connections, health checks, sticky sessions
- [ ] Do — run two app instances behind Nginx, watch traffic split
- [ ] Explain — what breaks when a stateful app is load balanced?

### 3. CDN `Concept`
- [ ] Learn — edge caching, cache headers, TTL, cache invalidation, origin
- [ ] Do — serve static assets through a CDN, compare load times
- [ ] Explain — what should never be cached at the edge?

### 4. Cloudflare `Tool`
- [ ] Learn — DNS, proxy mode, DDoS protection, WAF, page rules
- [ ] Do — point a real domain through Cloudflare
- [ ] Explain — what does "orange cloud" actually change?

**Milestone project**
- [ ] Nginx + two app instances + CDN in front, all working together

---

## Phase 2 — Making the Database Not Fall Over

### 5. Database Indexing `Concept`
- [ ] Learn — B-trees, composite indexes, covering indexes, reading `EXPLAIN`
- [ ] Do — million-row table, slow query, add index, measure the difference
- [ ] Explain — why does an index make writes slower?

### 6. Database Replication `Concept`
- [ ] Learn — primary/replica, sync vs async, replication lag, failover
- [ ] Do — set up a read replica, route reads to it
- [ ] Explain — what is the read-after-write problem?

### 7. Database Sharding `Concept`
- [ ] Learn — shard keys, range vs hash sharding, rebalancing, hotspots
- [ ] Do — sketch a sharding scheme for a real table you own
- [ ] Explain — why is sharding a last resort?

**Milestone project**
- [ ] Query that went from seconds to milliseconds, with before/after numbers written down

---

## Phase 3 — Scaling and Protecting the System

### 8. Vertical vs Horizontal Scaling `Concept`
- [ ] Learn — trade-offs, stateless services, session storage, sticky sessions
- [ ] Do — make your app fully stateless so any instance can serve any request
- [ ] Explain — what has to be true before you can scale horizontally?

### 9. Redis `Tool`
- [ ] Learn — data types, TTL, cache-aside pattern, persistence, eviction policies
- [ ] Do — cache your slowest endpoint, measure hit rate
- [ ] Explain — why is cache invalidation famously hard?

### 10. Rate Limiting `Concept`
- [ ] Learn — token bucket, leaky bucket, fixed vs sliding window
- [ ] Do — build a per-IP rate limiter with Redis counters
- [ ] Explain — where should rate limiting live: edge, gateway, or app?

**Milestone project**
- [ ] Cached, rate-limited, stateless API running on multiple instances

---

## Phase 4 — Real-Time Communication

### 11. Polling `Concept`
- [ ] Learn — short polling vs long polling, cost and latency trade-offs
- [ ] Do — build a live counter with short polling
- [ ] Explain — when is plain polling still the right answer?

### 12. Server-Sent Events `Concept`
- [ ] Learn — EventSource API, auto-reconnect, event IDs, connection limits
- [ ] Do — rebuild the same live counter with SSE
- [ ] Explain — why is SSE usually right for notifications and token streaming?

### 13. WebSockets `Concept`
- [ ] Learn — HTTP upgrade handshake, frames, heartbeats, scaling across instances
- [ ] Do — rebuild the live counter with WebSockets; try a small chat app
- [ ] Explain — what breaks when WebSocket servers sit behind a load balancer?

### 14. WebRTC `Concept`
- [ ] Learn — STUN, TURN, ICE, SDP, signaling, data channels
- [ ] Do — get a one-to-one video call working between two browsers
- [ ] Explain — why do you still need a server for a "peer-to-peer" protocol?

**Milestone project**
- [ ] Same feature built three ways — polling, SSE, WebSockets — with notes on the differences

---

## Phase 5 — Async Work and Messaging

### 15. Pub/Sub `Concept`
- [ ] Learn — topics, publishers, subscribers, fan-out, decoupling
- [ ] Do — publish and subscribe using Redis Pub/Sub
- [ ] Explain — what does Redis Pub/Sub lose that a real broker gives you?

### 16. RabbitMQ `Tool`
- [ ] Learn — exchanges, bindings, queues, acks, dead-letter queues, prefetch
- [ ] Do — move a slow task (email, image resize) to a background worker
- [ ] Explain — what happens to a message if the worker crashes mid-task?

### 17. Kafka `Tool`
- [ ] Learn — topics, partitions, offsets, consumer groups, retention, replay
- [ ] Do — produce and consume a stream; replay from an earlier offset
- [ ] Explain — how does the offset model differ from ack-and-delete?

**Milestone project**
- [ ] Request handler returns instantly; the slow work happens in a worker off a queue

---

## Phase 6 — Shipping It

### 18. Docker `Tool`
- [ ] Learn — images, layers, volumes, networks, multi-stage builds, Compose
- [ ] Do — containerize your app and its database with Docker Compose
- [ ] Explain — why is image layer order important for build speed?

### 19. CI/CD `Concept`
- [ ] Learn — pipeline stages, artifacts, secrets, environments, rollback strategy
- [ ] Do — GitHub Actions pipeline: test → build image → push
- [ ] Explain — what's the difference between continuous delivery and deployment?

### 20. Kubernetes `Tool`
- [ ] Learn — pods, deployments, services, ingress, configmaps, secrets
- [ ] Do — deploy to a local `kind` or `minikube` cluster
- [ ] Explain — how is an ingress different from a service?

**Milestone project**
- [ ] Push to `main` → tests run → image builds → app deploys, with no manual steps

---

## Phase 7 — Knowing What's Happening

### 21. Monitoring & Observability `Concept`
- [ ] Learn — logs, metrics, traces; RED and USE methods; SLIs and SLOs
- [ ] Do — add structured logging with request IDs
- [ ] Explain — what can observability answer that monitoring can't?

### 22. Prometheus `Tool`
- [ ] Learn — pull model, exporters, metric types, PromQL, alerting rules
- [ ] Do — expose `/metrics`, scrape it, write a PromQL query
- [ ] Explain — why pull-based instead of push?

### 23. Grafana `Tool`
- [ ] Learn — data sources, panels, variables, alerting
- [ ] Do — build a dashboard: request rate, error rate, p95 latency
- [ ] Explain — what belongs on a dashboard you'd actually look at during an incident?

**Milestone project**
- [ ] Live dashboard showing traffic, errors, and latency for your own app

---

## Phase 8 — Finding the Breaking Point

### 24. Load Testing `Concept`
- [ ] Learn — throughput vs latency, p50/p95/p99, ramp profiles, soak tests
- [ ] Do — run k6 against your API until latency degrades; record the number
- [ ] Explain — why are average latencies misleading?

### 25. Memory Leak `Concept`
- [ ] Learn — common causes, heap snapshots, profiling in your language
- [ ] Do — deliberately introduce a leak, then find it on your Grafana chart
- [ ] Explain — how do you tell a leak apart from normal cache growth?

**Milestone project**
- [ ] Written record of your system's breaking point and what failed first

---

## Phase 9 — Learn When You Actually Need Them

### 26. GraphQL `Concept`
- [ ] Learn — schema, resolvers, queries vs mutations, N+1 problem, DataLoader
- [ ] Do — wrap an existing REST API in a GraphQL layer
- [ ] Explain — when is REST still the better choice?

### 27. Turborepo `Tool`
- [ ] Learn — workspaces, task pipelines, local and remote caching
- [ ] Do — set up a two-package monorepo with a shared library
- [ ] Explain — what makes the second build faster than the first?

---

## Fast-Track Path

Short on time? These eight in order get most of the value:

- [ ] Nginx
- [ ] Database Indexing
- [ ] Redis
- [ ] Docker
- [ ] CI/CD
- [ ] Prometheus + Grafana
- [ ] Pub/Sub
- [ ] Load Testing

---

## Notes

Use this space for things you got stuck on, decisions you made, or topics to revisit.

```
Date        Topic                 Note
__________  ____________________  ________________________________________
__________  ____________________  ________________________________________
__________  ____________________  ________________________________________
__________  ____________________  ________________________________________
__________  ____________________  ________________________________________
```
