
Claude finished the response


Message collapsed

Categorized items into tools and concepts, opted for organized table format
Categorized items into tools and concepts, opted for organized table format
Tools (actual software you install/run/use)
Tool	What it is
Docker	Containerization runtime
Kubernetes	Container orchestration platform
Kafka / RabbitMQ	Message brokers (Kafka = distributed log, RabbitMQ = traditional queue)
Redis	In-memory data store (cache, queues, rate-limit counters, pub/sub)
Prometheus	Metrics collection + time-series database
Grafana	Dashboards/visualization on top of metric sources
Nginx	Web server, reverse proxy, load balancer
Cloudflare	CDN / DNS / DDoS + WAF provider
Concepts (ideas, patterns, protocols)
Concept	What it is
Rate Limiting	Capping request volume per client/window
Load Balancing	Distributing traffic across multiple servers
Server-Sent Events	One-way server→client streaming over HTTP
WebSockets	Full-duplex persistent connection protocol
Polling	Client repeatedly asking for updates (short/long polling)
GraphQL	Query language + spec for APIs
Database Sharding	Splitting data horizontally across nodes
Database Indexing	Data structures to speed up reads
Database Replication	Copying data to replicas for HA/read scaling
Vertical vs Horizontal Scaling	Bigger machine vs more machines
Memory Leak	Unreleased memory growing over time
Load Testing	Measuring behavior under simulated traffic
Two borderline ones worth flagging:

Load balancer — the concept is load balancing; the tools are Nginx, HAProxy, Envoy, AWS ALB. I put it under concepts since you listed Nginx separately.
GraphQL — it's a specification, not software. You implement it with tools like Apollo Server or Hasura.
Load testing likewise is a concept; k6, JMeter, and Locust are the tools.
Give me a MD file


Devised direct file creation strategy for markdown output
Devised direct file creation strategy for markdown output
Includes both tables, plus a section on the borderline cases (load balancer, GraphQL, load testing) where the tool/concept split isn't clean.


Tools vs concepts
Document · MD 
pubsub
webrtc
turborepo
Ci/CD
CDN
Monitoring and Observabiiity
Add this also


Categorized items as concepts or tools, identifying Turborepo alone
Categorized items as concepts or tools, identifying Turborepo alone
Of the six, only Turborepo is a pure tool — the rest are concepts with well-known tools implementing them, so I noted those pairings in the borderline section.


Tools vs concepts
Document · MD 

Want to be notified when Claude responds?





Claude is AI and can make mistakes. Please double-check responses.


Tools vs concepts · MD
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
| **Turborepo** | High-performance build system for JavaScript/TypeScript monorepos. Task orchestration with dependency-aware scheduling and remote caching. |
 
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
| **Pub/Sub** | Messaging pattern where publishers emit to a topic and subscribers receive independently. Neither side knows the other — decouples producers from consumers. |
| **WebRTC** | Protocol suite + browser API for peer-to-peer audio, video, and data. Uses STUN/TURN to traverse NATs; media flows directly between peers, not through your server. |
| **CI/CD** | Continuous Integration (merge + auto-test frequently) and Continuous Delivery/Deployment (auto-ship every passing build). A practice, backed by a pipeline. |
| **CDN** | Content Delivery Network — geographically distributed caches that serve static (and increasingly dynamic) content from a node near the user. |
| **Monitoring & Observability** | *Monitoring* = watching known metrics and alerting on known failure modes. *Observability* = being able to ask new questions about unknown failures from logs, metrics, and traces (the "three pillars"). |
 
---
 
## Borderline Cases Worth Flagging
 
Some items blur the line. Here's how to think about them:
 
- **Load balancer** — *Load balancing* is the concept; **Nginx**, **HAProxy**, **Envoy**, and **AWS ALB** are the tools that implement it. Listed under concepts here since Nginx is already called out separately.
- **GraphQL** — It's a specification, not runnable software. The tools that implement it are **Apollo Server**, **Hasura**, **GraphQL Yoga**, and **Relay** on the client side.
- **Load testing** — The concept; the tools are **k6**, **JMeter**, **Locust**, and **Gatling**.
- **Rate limiting** — The concept; commonly implemented *with* **Redis**, **Nginx**'s `limit_req` module, or **Cloudflare** rules.
- **Kafka vs RabbitMQ** — Both are tools, but they solve different problems. Kafka for event streaming and replay; RabbitMQ for task queues and complex routing.
- **Pub/Sub** — The *pattern* is a concept. But **Google Cloud Pub/Sub** is a specific managed product with that exact name, so context matters. Other implementations: **Redis Pub/Sub**, **Kafka**, **NATS**, **AWS SNS**.
- **CDN** — The *architecture* is a concept; **Cloudflare**, **Akamai**, **Fastly**, and **AWS CloudFront** are the providers.
- **CI/CD** — The *practice* is a concept; **GitHub Actions**, **GitLab CI**, **Jenkins**, **CircleCI**, and **ArgoCD** are the tools.
- **Monitoring & Observability** — The *discipline* is a concept; **Prometheus**, **Grafana**, **Datadog**, **OpenTelemetry**, **Jaeger**, and **Loki** are the tools.
- **WebRTC** — A W3C/IETF standard, so a concept — but every browser ships an implementation, and you'll still need a TURN server (**coturn**) and a signaling server, which are real tools you deploy.
---
 
## Quick Mental Model
 
> **Concepts** are the *what* and *why*. **Tools** are the *how*.
>
> You learn concepts once and they transfer everywhere. Tools change every few years — but knowing the concept underneath makes picking up a new tool fast.
 
