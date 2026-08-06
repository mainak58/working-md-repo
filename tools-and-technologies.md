# System Design Topics — Tools vs Concepts

A quick segregation of common backend/system-design topics into **Tools** (software you actually install, run, or subscribe to) and **Concepts** (ideas, patterns, protocols, and techniques).

---

## Tools

| Tool | What it is |
|---|---|
| **Docker** | Containerization runtime — packages an app with its dependencies into a portable image. |
| **Kubernetes** | Container orchestration platform — scheduling, scaling, self-healing, service discovery. |
| **Kafka** | Distributed commit log / event-streaming platform. High throughput, retains messages, consumers track their own offset. |
| **RabbitMQ** | Traditional message broker. Smart routing (exchanges, bindings), push-based delivery, messages ack'd and removed. |
| **Redis** | In-memory data store. Used for caching, session storage, rate-limit counters, leaderboards, pub/sub, and simple queues. |
| **Prometheus** | Metrics collection and time-series database. Pull-based scraping, PromQL query language, alerting rules. |
| **Grafana** | Visualization and dashboarding layer that sits on top of metric sources like Prometheus, Loki, or SQL databases. |
| **Nginx** | Web server, reverse proxy, load balancer, TLS terminator, and static file server. |
| **Cloudflare** | Edge platform — CDN, DNS, DDoS protection, WAF, and edge compute (Workers). |

---

## Concepts

| Concept | What it is |
|---|---|
| **Rate Limiting** | Capping how many requests a client can make in a time window. Algorithms: token bucket, leaky bucket, fixed window, sliding window. |
| **Load Balancing** | Distributing incoming traffic across multiple servers. Strategies: round-robin, least-connections, IP hash, weighted. |
| **Server-Sent Events (SSE)** | One-way server → client streaming over a single long-lived HTTP connection. Auto-reconnects, text-only. |
| **WebSockets** | Full-duplex, persistent, bidirectional connection. Starts as HTTP, upgrades to the `ws://` / `wss://` protocol. |
| **Polling** | Client repeatedly asks the server for updates. *Short polling* = fixed interval; *long polling* = server holds the request open until data exists. |
| **GraphQL** | A query language and specification for APIs. Client requests exactly the fields it needs; single endpoint instead of many REST routes. |
| **Database Sharding** | Splitting one dataset horizontally across multiple database nodes by a shard key. Scales writes and storage. |
| **Database Indexing** | Auxiliary data structures (usually B-trees or hash indexes) that speed up reads at the cost of slower writes and extra storage. |
| **Database Replication** | Copying data from a primary to one or more replicas for high availability, failover, and read scaling. |
| **Vertical vs Horizontal Scaling** | *Vertical* = bigger machine (more CPU/RAM), simple but capped. *Horizontal* = more machines, near-unlimited but needs coordination. |
| **Memory Leak** | Memory that's allocated but never released, so usage grows over time until the process degrades or crashes. |
| **Load Testing** | Measuring how a system behaves under simulated traffic — throughput, latency percentiles, and breaking point. |

---

## Borderline Cases Worth Flagging

Some items blur the line. Here's how to think about them:

- **Load balancer** — *Load balancing* is the concept; **Nginx**, **HAProxy**, **Envoy**, and **AWS ALB** are the tools that implement it. Listed under concepts here since Nginx is already called out separately.
- **GraphQL** — It's a specification, not runnable software. The tools that implement it are **Apollo Server**, **Hasura**, **GraphQL Yoga**, and **Relay** on the client side.
- **Load testing** — The concept; the tools are **k6**, **JMeter**, **Locust**, and **Gatling**.
- **Rate limiting** — The concept; commonly implemented *with* **Redis**, **Nginx**'s `limit_req` module, or **Cloudflare** rules.
- **Kafka vs RabbitMQ** — Both are tools, but they solve different problems. Kafka for event streaming and replay; RabbitMQ for task queues and complex routing.

---

## Quick Mental Model

> **Concepts** are the *what* and *why*. **Tools** are the *how*.
>
> You learn concepts once and they transfer everywhere. Tools change every few years — but knowing the concept underneath makes picking up a new tool fast.
