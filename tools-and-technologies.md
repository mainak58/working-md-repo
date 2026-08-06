# System Design Roadmap — Tools vs Concepts, in Learning Order

Everything below is grouped into **phases**. Each phase builds on the one before it, so work top to bottom. Every item is tagged as a **Concept** (an idea, pattern, or protocol) or a **Tool** (software you install, run, or subscribe to).

> **The one rule that matters:** learn the *concept* before the *tool*. The concept tells you what problem exists and why. The tool is just one vendor's answer to it. Concepts last decades; tools get replaced every few years.

---

## Phase 0 — Prerequisites

Don't start the roadmap until these are comfortable. If they aren't, the rest will feel like memorizing trivia.

- How HTTP works — methods, status codes, headers, cookies
- What a client, a server, and a port actually are
- Basic Linux command line
- One backend language and one database you can build a small CRUD app in
- Git

**Build first:** a small REST API with a database behind it, running on your own machine. Everything after this is about making that app faster, more reliable, and able to handle more people.

---

## Phase 1 — Getting Traffic to Your Server

The first thing you hit in the real world: something has to sit between the internet and your app.

| # | Topic | Type | Why here |
|---|---|---|---|
| 1 | **Nginx** | Tool | Your first reverse proxy. Serves static files, terminates TLS, forwards requests to your app. Learn this before load balancers — it *is* one. |
| 2 | **Load Balancing** | Concept | One server isn't enough. Learn round-robin, least-connections, health checks, and sticky sessions. Nginx already does this, so it's a natural next step. |
| 3 | **CDN** | Concept | Caching static assets at edge locations near users. The cheapest performance win that exists. |
| 4 | **Cloudflare** | Tool | The CDN concept made concrete, plus DNS, DDoS protection, and a WAF. Free tier is generous — set it up on a real domain. |

**Build:** put Nginx in front of your app, run two copies of the app behind it, and watch traffic split between them.

---

## Phase 2 — Making the Database Not Fall Over

Databases are where most real systems actually break. Learn these in this exact order — each one assumes the previous.

| # | Topic | Type | Why here |
|---|---|---|---|
| 5 | **Database Indexing** | Concept | Start here. 90% of "our app is slow" is a missing index. Learn B-trees, composite indexes, and how to read an `EXPLAIN` output. |
| 6 | **Database Replication** | Concept | Copy data to replicas for failover and read scaling. Introduces replication lag and eventual consistency — the first real distributed-systems idea you'll meet. |
| 7 | **Database Sharding** | Concept | Splitting data across nodes by a shard key. Learn it *last* of the three, and understand that it's a last resort — it makes joins, transactions, and rebalancing genuinely hard. |

**Build:** load a table with a million rows, run a slow query, add an index, measure the difference yourself.

---

## Phase 3 — Scaling and Protecting the System

Now that you can scale a database, learn the vocabulary for scaling everything else.

| # | Topic | Type | Why here |
|---|---|---|---|
| 8 | **Vertical vs Horizontal Scaling** | Concept | The framing for every scaling decision. Bigger machine vs more machines, and why stateless services are what make horizontal scaling possible. |
| 9 | **Redis** | Tool | Your first cache. Also covers cache invalidation, TTLs, and the cache-aside pattern — the concepts that make it useful. |
| 10 | **Rate Limiting** | Concept | Protects everything you just built. Learn token bucket and sliding window, then implement one with Redis counters. |

**Build:** add a Redis cache in front of your slowest endpoint, then add a per-IP rate limiter using Redis.

---

## Phase 4 — Real-Time Communication

A clean progression from simplest to hardest. Each step exists because the previous one wasn't enough.

| # | Topic | Type | Why here |
|---|---|---|---|
| 11 | **Polling** | Concept | The naive solution — client asks repeatedly. Learn short polling, then long polling. Understanding why it's wasteful motivates everything below. |
| 12 | **Server-Sent Events (SSE)** | Concept | One-way server → client streaming over plain HTTP. Auto-reconnects, dead simple. Correct answer for notifications, live feeds, and AI token streaming. |
| 13 | **WebSockets** | Concept | Full-duplex persistent connection. Reach for this only when the client also needs to push — chat, multiplayer, collaborative editing. |
| 14 | **WebRTC** | Concept | Peer-to-peer audio, video, and data. Hardest of the four by a wide margin: STUN, TURN, ICE, SDP, and you still need a signaling server. Skip until you specifically need P2P media. |

**Build:** the same live-counter feature three times — polling, then SSE, then WebSockets. The differences become obvious immediately.

---

## Phase 5 — Async Work and Messaging

The moment you have a task too slow for a request cycle, you need this.

| # | Topic | Type | Why here |
|---|---|---|---|
| 15 | **Pub/Sub** | Concept | The decoupling pattern — publishers emit to a topic, subscribers consume independently, neither knows the other. Try it in Redis first; it's ten lines of code. |
| 16 | **RabbitMQ** | Tool | Traditional broker. Push-based, messages acknowledged and removed, smart routing via exchanges. The right mental model for *task queues*. |
| 17 | **Kafka** | Tool | Distributed commit log. Messages are retained and replayable, consumers track their own offset. The right mental model for *event streams*. Learn after RabbitMQ — the offset model only makes sense by contrast. |

**Build:** move email sending out of your request handler and into a background worker consuming from a queue.

---

## Phase 6 — Shipping It

You now have a system worth deploying properly.

| # | Topic | Type | Why here |
|---|---|---|---|
| 18 | **Docker** | Tool | Package the app with its dependencies. Learn images, layers, volumes, networks, and Docker Compose before touching anything else in this phase. |
| 19 | **CI/CD** | Concept | Continuous Integration (merge and test constantly) and Continuous Delivery (ship every passing build). Implement it with GitHub Actions — build the image, run tests, push it. |
| 20 | **Kubernetes** | Tool | Container orchestration. Genuinely hard, and pointless without solid Docker fundamentals. Learn pods, deployments, services, and ingress — and note that ingress is just the load balancing from Phase 1 again. |

**Build:** containerize your app, wire up a pipeline that tests and builds on every push, then deploy it to a local `kind` or `minikube` cluster.

---

## Phase 7 — Knowing What's Happening

Never skip this. You cannot fix what you cannot see.

| # | Topic | Type | Why here |
|---|---|---|---|
| 21 | **Monitoring & Observability** | Concept | *Monitoring* watches known metrics for known failures. *Observability* lets you ask new questions about failures nobody predicted. Learn the three pillars: logs, metrics, traces. |
| 22 | **Prometheus** | Tool | Metrics collection and time-series storage. Pull-based scraping, PromQL, alerting rules. Instrument your own app with a client library. |
| 23 | **Grafana** | Tool | Dashboards on top of Prometheus. Always learn it *after* Prometheus — Grafana only visualizes data something else collected. |

**Build:** expose a `/metrics` endpoint, scrape it with Prometheus, and chart request latency in Grafana.

---

## Phase 8 — Finding the Breaking Point

With dashboards in place, you can finally measure instead of guess.

| # | Topic | Type | Why here |
|---|---|---|---|
| 24 | **Load Testing** | Concept | Simulated traffic to find throughput limits and latency percentiles. Focus on p95/p99, not averages. Tools: k6, JMeter, Locust. |
| 25 | **Memory Leak** | Concept | Memory allocated but never released, growing until the process dies. You'll usually spot it as a slow upward slope on a Grafana chart — which is exactly why this comes after Phase 7. |

**Build:** run k6 against your API and push it until latency degrades. Find the number.

---

## Phase 9 — Learn When You Actually Need Them

Genuinely useful, but solving problems a beginner doesn't have yet. Don't front-load these.

| # | Topic | Type | When you need it |
|---|---|---|---|
| 26 | **GraphQL** | Concept | When REST endpoint sprawl or client over-fetching becomes a real pain. Understand REST properly first, or you won't see what problem it solves. |
| 27 | **Turborepo** | Tool | When you have several JS/TS packages in one repo and builds get slow. Pure developer-experience tooling, unrelated to system design. |

---

## Fast-Track: The Minimum Path

If you only have limited time, this ordered subset gets you most of the value:

`Nginx → Indexing → Redis → Docker → CI/CD → Prometheus + Grafana → Pub/Sub → Load Testing`

---

## Reference: Full Split

**Tools (9):** Nginx · Cloudflare · Redis · RabbitMQ · Kafka · Docker · Kubernetes · Prometheus · Grafana · Turborepo

**Concepts (17):** Load Balancing · CDN · Database Indexing · Database Replication · Database Sharding · Vertical vs Horizontal Scaling · Rate Limiting · Polling · SSE · WebSockets · WebRTC · Pub/Sub · CI/CD · Monitoring & Observability · Load Testing · Memory Leak · GraphQL

---

## Borderline Cases

Several items sit on the line. The pattern is almost always *concept, implemented by tools*:

| Item | Concept | Tools that implement it |
|---|---|---|
| **Load balancer** | Load balancing | Nginx, HAProxy, Envoy, AWS ALB |
| **CDN** | Edge caching architecture | Cloudflare, Akamai, Fastly, CloudFront |
| **CI/CD** | The practice | GitHub Actions, GitLab CI, Jenkins, ArgoCD |
| **Pub/Sub** | The messaging pattern | Redis Pub/Sub, Kafka, NATS, AWS SNS, Google Cloud Pub/Sub |
| **Monitoring & Observability** | The discipline | Prometheus, Grafana, OpenTelemetry, Jaeger, Datadog |
| **Load testing** | The practice | k6, JMeter, Locust, Gatling |
| **GraphQL** | A specification | Apollo Server, Hasura, GraphQL Yoga |
| **Rate limiting** | The technique | Redis counters, Nginx `limit_req`, Cloudflare rules |
| **WebRTC** | A W3C/IETF standard | Browser APIs, plus coturn for TURN and your own signaling server |

Note that **Google Cloud Pub/Sub** is a specific product that shares a name with the generic pattern — context tells you which one someone means.

---

## Two Notes on Method

**Kafka and RabbitMQ are not competitors.** Kafka is a retained, replayable event log. RabbitMQ is a task queue with smart routing. Choosing between them is a design decision, not a preference.

**Build something at every phase.** Reading about sharding teaches you the word. Actually watching replication lag break your read-after-write assumption teaches you the concept.
