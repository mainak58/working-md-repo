# Rate Limiting — End-to-End Conceptual Guide

*Concepts only. No framework code. Focus: **why** it exists, **when** each approach fits, and **where people get it wrong**.*

---

## 1. What rate limiting actually is

Rate limiting is **admission control**: deciding, before you spend resources on a request, whether you are willing to spend them at all.

That framing matters more than the algorithm. Everything else in this document follows from it.

### Terms people mix up

| Term | What it really means | Failure it addresses |
|---|---|---|
| **Rate limiting** | Cap on *events per unit time* (e.g. 100 req/min) | Sustained overuse |
| **Concurrency limiting** | Cap on *simultaneous in-flight work* (e.g. 20 open requests) | Resource exhaustion, queue buildup |
| **Throttling** | Slowing a caller down (delay) rather than rejecting | Smoothing bursts |
| **Policing** | Rejecting excess outright | Hard protection |
| **Shaping** | Buffering excess and releasing at a steady rate | Smoothing, at the cost of latency |
| **Quota** | Cap over a long billing/business period (10k calls/month) | Cost and commercial tiers |
| **Load shedding** | Dropping work when the *system* is unhealthy, regardless of who sent it | Overload survival |
| **Backpressure** | Signalling upstream to slow down, propagated through the call chain | Systemic stability |
| **Circuit breaking** | Stopping calls to a *dependency* that's failing | Cascading failure |

**Rate limiting is caller-centric. Load shedding is system-centric.** You need both. A rate limiter says "you've had enough"; a load shedder says "*everyone* has had enough right now." A system with only rate limits still dies when every caller stays under their limit simultaneously.

---

## 2. Why rate limit — the real reasons

Most docs list "prevent abuse" and stop. The actual motivations are distinct, and **they imply different designs**. Know which one you're solving for.

### 2.1 Protect finite capacity
Your service has a real ceiling — CPU, DB connections, thread pool, downstream API quota. Beyond it, latency doesn't degrade gracefully, it *collapses* (queues grow, timeouts fire, retries pile on, throughput goes to zero). A limiter keeps you on the good side of the knee in the curve.

### 2.2 Fairness / noisy neighbor
In multi-tenant systems the default failure mode is: one customer's batch job consumes everything, everyone else sees timeouts. Per-tenant limits convert a **global outage** into **one unhappy tenant**.

### 2.3 Cost control
When each request costs real money (LLM tokens, third-party APIs, egress, cloud functions), the limit isn't protecting capacity, it's protecting a budget. Different design: cost-weighted, long windows, hard stops.

### 2.4 Abuse and security
Credential stuffing, scraping, enumeration, spam, DoS. Here the adversary is *adaptive* — they will rotate whatever key you limit on. This is the only category where the caller is actively hostile, and it drives very different key choices.

### 2.5 Commercial tiering
Free = 100/day, Pro = 10k/day. Here the limit is a *product feature*, not an engineering safeguard. It must be predictable, documented, and precisely enforced — customers will build against it.

### 2.6 Preventing cascading failure
Limits create firewalls between services so that a surge in one component doesn't propagate into a full-system brownout. Combined with timeouts, retries with budgets, and circuit breakers.

> **Key insight:** #1 and #6 want *adaptive* limits based on live system health. #5 wants *fixed, contractual* limits that never surprise a customer. Trying to serve both with one mechanism is a common root cause of pain.

---

## 3. The design questions, in order

Answer these before choosing an algorithm. The algorithm is the least interesting decision.

1. **What resource am I actually protecting?** (CPU? a DB? a vendor bill? a downstream partner?)
2. **What unit best represents consumption?** (requests? rows scanned? tokens? bytes? seconds of CPU?)
3. **Who is the subject of the limit?** (IP? user? API key? tenant? endpoint? all of them, layered?)
4. **What happens when the limit is hit?** (reject? delay? degrade? queue? shed by priority?)
5. **What happens when the limiter itself fails?** (fail open or fail closed — decide deliberately)
6. **How does the caller find out and adapt?** (headers, error semantics, documentation)

---

## 4. Choosing the key (the dimension you limit on)

The key determines who gets punished. Getting this wrong is more damaging than picking a suboptimal algorithm.

| Key | Good for | Breaks when |
|---|---|---|
| IP address | Anonymous traffic, pre-auth endpoints | NAT/CGNAT/corporate proxies (one office = one IP), mobile carriers, IPv6 (attacker has /64 = 18 quintillion addresses), cloud attackers rotating IPs cheaply |
| User ID / account | Authenticated fairness | Attacker creates many accounts; doesn't protect pre-login endpoints |
| API key / client ID | B2B tiering, billing | Key shared across a customer's whole fleet; leaked keys |
| Tenant / org | Multi-tenant fairness | One tenant is legitimately 1000x bigger — flat limits are unfair either way |
| Endpoint / route | Protecting one expensive operation | Ignores that a caller may hammer many cheap endpoints at once |
| Resource ID | Hot-object protection (one document, one seat inventory) | Very high cardinality |
| Global | Absolute capacity ceiling | Alone, it's first-come-first-served — the loudest caller wins |
| Composite (tenant + endpoint) | Precision | Cardinality explosion, memory cost, harder to explain to customers |

**Practical stance:** use *layers*, not one key. A common, sane stack:

```
Global ceiling (protect the system)
  └─ Per-tenant limit (fairness)
       └─ Per-endpoint limit within tenant (protect expensive routes)
            └─ Per-IP limit on unauthenticated routes (abuse)
```

The first layer that says no, wins. Each layer answers a different question.

---

## 5. Algorithms — what they do, and honestly when to use each

### 5.1 Fixed window counter
Count requests in a wall-clock bucket (e.g. per minute); reset at boundary.

- **Why:** trivially cheap, one counter, one TTL. Easy for humans to reason about.
- **When:** coarse quotas ("1M calls/month"), low-stakes internal limits, when boundary bursts are harmless.
- **Weakness:** the **boundary burst** — a caller can send the full allowance at 11:59:59 and again at 12:00:00, i.e. **2× the intended rate** in a moment straddling the boundary. Also causes synchronized stampedes: everyone waits for the top of the minute.

### 5.2 Sliding window log
Store the timestamp of every request; count those within the trailing window.

- **Why:** exactly correct, no boundary artifacts.
- **When:** low volume, high value — expensive operations, security-sensitive endpoints, where precision matters more than memory.
- **Weakness:** memory and cost scale with request volume. Not viable at high QPS for many keys.

### 5.3 Sliding window counter (approximate)
Blend the current and previous fixed windows by how far you are into the current one.

- **Why:** ~fixed-window cost, ~sliding-window accuracy. Small, bounded error.
- **When:** the pragmatic default for high-volume HTTP APIs. This is what most large gateways actually do.
- **Weakness:** approximate; assumes traffic in the previous window was evenly distributed.

### 5.4 Token bucket
A bucket refills at a steady rate up to a capacity; each request removes tokens. Empty bucket = reject or wait.

- **Why:** it separates **average rate** (refill) from **burst tolerance** (capacity), which matches how real clients behave — idle, then a flurry.
- **When:** public APIs, anything with bursty-but-legitimate clients, cost-weighted limiting (a request can cost N tokens).
- **Weakness:** two parameters to tune, and a burst of size = capacity can arrive instantly. Choose capacity by asking "what's the largest burst my system can absorb without harm?"

### 5.5 Leaky bucket (as a queue)
Requests enter a fixed-size queue and drain at a constant rate; overflow is dropped.

- **Why:** produces a perfectly smooth output rate — protects fragile downstreams that hate bursts.
- **When:** shaping traffic toward a legacy system, batch pipelines, outbound calls to a partner with strict pacing.
- **Weakness:** adds latency by design; the queue is a place where requests go to time out. Never make the queue unbounded.

### 5.6 GCRA (Generic Cell Rate Algorithm)
Token-bucket-equivalent behavior stored as a single "theoretical arrival time" value — no periodic refill, no timers.

- **Why:** O(1) memory per key, no background jobs, precise, easy to implement atomically in a shared store.
- **When:** you need token-bucket semantics at scale in a distributed store.
- **Weakness:** less intuitive to explain and debug.

### 5.7 Concurrency limiter (semaphore / bulkhead)
Cap the number of requests in flight, not the arrival rate.

- **Why:** **the thing that kills servers is usually concurrency, not RPS.** 100 req/s at 10 ms each = 1 concurrent request. 100 req/s at 5 s each = 500 concurrent — thread pools exhausted, memory gone. A rate limit cannot see this; a concurrency limit can.
- **When:** protecting thread pools, DB connections, memory-heavy work, any slow or variable-latency operation. **Almost every service should have one, even if it also has rate limits.**
- **Weakness:** doesn't express fairness or business quotas on its own.

### 5.8 Adaptive / dynamic limiting (AIMD, latency-based)
Continuously adjust the allowed concurrency based on observed latency or error rate — the same congestion-control logic TCP uses. Additive increase while healthy, multiplicative decrease on distress.

- **Why:** you don't have to guess the magic number, and it tracks capacity as it changes (deploys, degraded nodes, noisy hardware, cold caches).
- **When:** internal service-to-service traffic, and as the outermost safety net in front of static limits.
- **Weakness:** harder to reason about; can oscillate if poorly tuned; unsuitable as a *contractual* customer-facing limit because it isn't predictable.

### 5.9 Priority-based load shedding
When overloaded, drop the least important work first (health checks and payments live; recommendation prefetch dies). Often driven by **queue latency** (CoDel-style) rather than a fixed threshold.

- **Why:** during overload, *what* you drop matters more than *how much*.
- **When:** any system with mixed criticality traffic. Requires request classification — do this early, it's hard to retrofit.

### Quick selection table

| Situation | Reach for |
|---|---|
| Public API with tiers | Token bucket or sliding window counter, per API key |
| Monthly billing quota | Fixed window (long period) + cost weighting |
| Login / OTP / password reset | Sliding window log, **layered keys** (IP *and* account *and* global) |
| Protecting a thread pool or DB | Concurrency limiter |
| Slow or variable-latency work | Concurrency limiter, not RPS |
| Internal microservice calls | Adaptive concurrency + retry budgets |
| Fragile downstream needing smooth flow | Leaky bucket (bounded queue) |
| Expensive/variable-cost calls (LLM tokens) | Token bucket with cost-weighted debits |
| Overload survival | Priority shedding + latency-based shedding |

---

## 6. Where to enforce it

Enforcement location is a trade between *cheapness of rejection* and *quality of information*.

| Layer | Sees | Best at | Blind to |
|---|---|---|---|
| Client SDK | Its own behavior only | Politeness, retry backoff, self-throttling | Everyone else |
| CDN / edge | IP, geography, raw volume | Cheap absorption of volumetric attacks | Identity, cost of a request |
| API gateway | Auth identity, route | Per-key/tenant/route limits — the usual home | Actual internal resource pressure |
| Service mesh / sidecar | Service-to-service calls | Concurrency limits, bulkheads, circuit breaking | Business identity |
| Application | Everything — user, plan, request cost | Cost-weighted and business-rule limits | Traffic it never sees (already at capacity to reject) |
| Data layer | Query cost | Last-resort protection (statement timeouts, connection caps) | Everything else |

**Rule of thumb:** reject as early as possible, but decide as late as necessary. Cheap coarse limits at the edge; precise, expensive, identity-aware limits deeper in. A limit enforced *after* the expensive work is not a limit, it's a metric.

---

## 7. Distributed rate limiting

Once you have more than one server, "100 req/min" becomes ambiguous.

### Option A — Local limits (per instance)
Divide the global limit by instance count.

- **Pros:** zero latency, zero dependencies, cannot fail.
- **Cons:** wrong under uneven load balancing; wrong the moment you autoscale; a caller pinned to one node gets 1/N of the intended allowance while the fleet is idle.
- **Use when:** approximate protection is fine and availability is paramount.

### Option B — Centralized counter (shared store, e.g. Redis)
All instances increment the same counter atomically.

- **Pros:** accurate, consistent, simple mental model.
- **Cons:** adds a network hop to *every* request; the store becomes a hot shard and a single point of failure; needs an explicit fail-open/fail-closed policy.
- **Use when:** correctness matters (billing, security) and you can afford the hop.

### Option C — Local enforcement with async global reconciliation
Each node enforces locally against a share of the budget, periodically syncing usage and rebalancing shares.

- **Pros:** fast path stays local; converges to global correctness; degrades gracefully.
- **Cons:** briefly permissive during bursts; more moving parts.
- **Use when:** high volume plus a need for roughly-global fairness. This is what most large-scale systems land on.

### Option D — Sticky routing
Hash the limit key so all requests for a given tenant land on the same node, making the local counter authoritative.

- **Pros:** accurate and fast.
- **Cons:** hot keys create hot nodes; rebalancing on scaling events resets state; couples routing to limiting.

### Cross-cutting concerns
- **Clock skew** between nodes shifts window boundaries. Prefer the store's clock or monotonic time; never trust distributed wall clocks for tight windows.
- **Atomicity:** read-modify-write across the network races. Use atomic operations/scripts, or accept the overcount.
- **Fail-open vs fail-closed:** if the limiter's store is down, do you allow everything (risking overload) or block everything (guaranteeing an outage)? Usually: **fail open with a conservative local fallback limit.** Whatever you choose, choose it explicitly and test it.

---

## 8. What to return, and how clients should react

The protocol between limiter and client is half the design, and the half most often neglected.

### Status codes
- **429 Too Many Requests** — *you*, the caller, exceeded your allowance. Deterministic; retrying later with the same pattern will fail again.
- **503 Service Unavailable** — *the system* is overloaded or shedding. Not the caller's fault; retrying later may well succeed.

Distinguishing these lets clients (and your own dashboards) behave correctly. Collapsing everything into 429 teaches clients the wrong lesson.

### Signalling
- `Retry-After` — how long to wait. **Add jitter server-side or instruct clients to jitter**, otherwise every rejected client returns at the same instant.
- Rate-limit headers (limit / remaining / reset) let well-behaved clients self-pace *before* being rejected. This is the single highest-leverage thing you can do to reduce 429s: make the budget visible.
- Document limits publicly, including burst capacity and how cost weighting works. Undocumented limits become support tickets and angry blog posts.

### Client-side responsibilities
- **Exponential backoff with full jitter.** Backoff without jitter just synchronizes the herd.
- **Retry budgets** — cap retries as a *percentage of total traffic* (e.g. retries ≤ 10% of requests), not as a per-request count. Per-request counts are what create retry storms during a partial outage.
- **Respect `Retry-After`.** Don't retry a 429 immediately — that's the behavior the limit exists to stop.
- **Circuit breakers** — after sustained rejection, stop calling for a while rather than grinding.
- **Idempotency keys** so retries are safe and don't double-charge the limit or the customer.

---

## 9. Where most people get it wrong

This is the section that matters. Nearly every incident I've seen involving rate limiting traces to one of these.

### 9.1 Counting requests when costs vary by orders of magnitude
`GET /health` and `POST /report/generate` both count as "1". A caller stays comfortably under 100 req/min while destroying the database.
**Fix:** weight by cost. Debit tokens proportional to expected (or measured) work — rows scanned, bytes returned, tokens generated, CPU seconds. Charge an estimate up front, reconcile the true cost afterward.

### 9.2 Confusing rate with concurrency
The classic outage: RPS is normal, latency of a downstream triples, in-flight requests explode, thread pool exhausts, service dies — while the rate limiter reports everything is fine.
**Fix:** concurrency limits are usually the more important protection. Add them even when you already have rate limits.

### 9.3 Limiting by IP without thinking about who shares IPs
One office, one university, or one mobile carrier behind CGNAT looks like a single abusive client. Meanwhile a real attacker rents a /64 of IPv6 or a residential proxy pool and never hits your per-IP limit.
**Fix:** IP limits are a blunt anti-volumetric tool only. Bucket IPv6 by prefix (/64 or /56), keep per-IP limits generous, and put real limits on authenticated identity.

### 9.4 Login endpoints keyed on only one dimension
- Key on IP only → credential stuffing from a rotating proxy pool sails through.
- Key on username only → an attacker locks out any account they choose (a DoS you built yourself).
**Fix:** layer them — per-IP, per-account, per-IP+account, and a global anomaly threshold. Use progressive friction (delay, CAPTCHA, step-up auth) rather than a hard lockout.

### 9.5 Retry amplification
Client retries 3×, gateway retries 3×, service retries 3× → **one user action becomes 27 requests**, and they all arrive precisely when the system is already struggling. Rate limiting *causes* the overload it was meant to prevent.
**Fix:** retry at exactly one layer. Use retry budgets. Never retry a 429 without honoring backoff. Mark retried requests (a header) so limiters and dashboards can see them.

### 9.6 No jitter → synchronized stampedes
Every client is told "retry after 60s", so at t+60s you receive the entire rejected population at once. Fixed windows do this too: the reset boundary becomes a scheduled thundering herd.
**Fix:** jitter everything — retry delays, window boundaries (offset per key by a hash), background job schedules.

### 9.7 Setting limits by guesswork
Numbers picked in a meeting ("1000/min sounds right"), never validated. Two outcomes: the limit is so high it never triggers before the system falls over (decorative), or so low it blocks legitimate use (support burden).
**Fix:** load test to find the actual knee in the latency curve, set the limit meaningfully below it, then measure the p99 of *real* customer usage to confirm you're not clipping legitimate traffic. Revisit after significant capacity or code changes.

### 9.8 Enforcing the limit after doing the work
Authenticating, parsing, hydrating, querying — and *then* checking the limit. You've paid the full cost and returned an error for it.
**Fix:** check as early as identity allows. Cheap checks (IP, global) before auth; identity checks immediately after auth and before any real work.

### 9.9 Unbounded queues in front of the limiter
"We won't reject, we'll just queue." The queue grows, latency climbs past client timeouts, and you spend 100% of capacity serving responses nobody is waiting for any more. **Throughput becomes zero while utilization is 100%.**
**Fix:** bounded queues, always. Shed on overflow. Drop requests whose deadline has already passed. Prefer fast rejection over slow success.

### 9.10 Not deciding what happens when the limiter breaks
Redis hiccups and either (a) everything 500s because the limiter throws, or (b) all limits vanish silently and the fleet gets flattened.
**Fix:** an explicit policy, with a conservative in-process fallback limit, plus an alert. Test it — kill the store in staging and watch.

### 9.11 One global limit and nothing else
First-come-first-served means the noisiest caller consumes the shared budget and everyone else is starved. Technically "protected," commercially a disaster.
**Fix:** per-tenant fairness underneath the global ceiling.

### 9.12 Per-tenant limits that sum to far more than capacity
1,000 tenants × 100 req/s each = 100,000 req/s against a system that handles 5,000. Every tenant is within their limit while the service is on fire.
**Fix:** the global ceiling and load shedding still have to exist. Per-tenant limits provide fairness, not capacity protection. Model the aggregate, not just the individual.

### 9.13 Treating rate limiting as a substitute for capacity planning or authorization
A limiter is not a scaling strategy, and "1 req/hour" is not access control. If legitimate demand exceeds capacity, the answer is more capacity or a different architecture — throttling everyone just distributes the pain.

### 9.14 Rate limiting the wrong traffic
Health checks throttled → load balancer removes healthy nodes → cascading failure. Internal control-plane traffic throttled during an incident → you can't deploy the fix. Payment callbacks throttled → lost revenue and reconciliation nightmares.
**Fix:** classify traffic by criticality. Exempt or separately-budget health checks, control plane, and webhooks from partners. Never let a limit block your ability to respond to an incident.

### 9.15 Silent limits — no headers, no docs, no signal
Clients can't self-regulate against a budget they can't see, so they discover it by failing in production, then hammer you with support tickets.
**Fix:** publish limits, emit remaining-budget headers, and warn (log, email, dashboard) before a customer starts hitting a wall.

### 9.16 429s hidden in error dashboards
Either 429s are lumped into "5xx errors" and drown out real failures, or they're filtered out entirely and nobody notices a customer has been blocked for three days.
**Fix:** track them as their own signal, dimensioned by key. Alert on *sustained* rejection for any single tenant — that's usually either a broken integration or a customer you're about to lose.

### 9.17 Shaping in a place that costs resources
"We don't reject, we sleep for 200 ms." Every sleeping request still holds a thread, socket, and memory. Under load you've converted a rate problem into a concurrency problem.
**Fix:** delay only where it's cheap (async/event-loop, edge). Otherwise reject and let the client back off.

### 9.18 Boundary artifacts from fixed windows
The 2× burst at window edges (see §5.1) is genuinely surprising in production: monitoring shows a limit of 100/min while 200 requests land within two seconds.
**Fix:** sliding window counter or token bucket for anything where the burst matters.

### 9.19 Forgetting bulk and batch endpoints
`POST /users/batch` with 10,000 items counts as one request. So does a GraphQL query requesting the entire object graph, or a WebSocket connection that then streams thousands of messages.
**Fix:** limit on items/complexity, not envelopes. For GraphQL, use query complexity/depth analysis. For long-lived connections, limit both connections and messages within them.

### 9.20 Ignoring the limiter's own cost and cardinality
A composite key of tenant × endpoint × method × IP produces millions of counters, and your Redis memory (or the extra network hop) becomes the new bottleneck. The limiter takes down the service.
**Fix:** keep cardinality bounded and deliberate, expire keys aggressively, and budget the limiter's own latency and memory as a first-class concern.

### 9.21 Static limits in an elastic system
Limits tuned when you ran 10 nodes are silently wrong at 50 — either you never protect anything or you throttle at a fraction of real capacity. Same problem after a deploy that changes per-request cost.
**Fix:** express limits as a fraction of measured capacity, or use adaptive limiting for the protection layer while keeping static limits only for contractual tiers.

### 9.22 Leaking information through the limiter
Different rejection behavior for existing vs non-existent accounts turns your login limiter into a user-enumeration oracle. Detailed limit responses can also tell an attacker exactly how fast they can go without tripping detection.
**Fix:** uniform responses on security-sensitive endpoints; be generous with detail on business APIs, stingy on auth ones.

---

## 10. How to actually choose the numbers

1. **Measure capacity.** Load test to find where p99 latency turns upward. That knee, not the point of total failure, is your ceiling.
2. **Set the global limit below the knee.** Leave headroom for retries, background work, and degraded nodes.
3. **Look at real usage distributions.** Plot per-tenant request rates. Set the per-tenant limit above the p99 of legitimate use — you're clipping outliers, not the mainstream.
4. **Pick burst capacity from the workload.** How large a burst can you absorb without exceeding the knee? That's your token bucket size.
5. **Deploy in observe-only mode first.** Log what *would* have been rejected, for days, across a full weekly cycle. This step catches almost every bad limit before customers do.
6. **Roll out gradually** — a small traffic percentage, then a few tenants, then everyone.
7. **Re-evaluate on a schedule** and after capacity or cost-per-request changes.

---

## 11. Observability checklist

Track, dimensioned by limit key and layer:

- Allowed vs rejected counts, and rejection *rate*
- Rejections per tenant — spot the broken integration and the customer in trouble
- Remaining-budget distribution — who is at 95% and about to be a problem
- Limiter decision latency and store error rate
- Concurrency and queue depth (usually more diagnostic than RPS)
- Retry ratio — how much of your traffic is retries
- What *would* have been rejected by proposed limit changes (shadow evaluation)

Alert on: sustained rejection for a single tenant, sudden global rejection spikes, limiter store errors, and retry ratio climbing.

---

## 12. One-page summary

- Rate limiting is **admission control**, not a scaling strategy and not authorization.
- Know **which of the six motivations** you're serving — protection, fairness, cost, abuse, tiering, cascade prevention. They imply different designs.
- The **key** matters more than the algorithm. Layer keys: global → tenant → endpoint → IP.
- **Concurrency limits protect servers; rate limits protect budgets and fairness.** Most systems need both.
- **Token bucket** for bursty public APIs, **sliding window counter** for high-volume accuracy, **sliding window log** for low-volume precision, **adaptive concurrency** for internal traffic, **priority shedding** for overload.
- Enforce **early and cheaply at the edge, precisely and late in the app**. Never after the expensive work.
- Distributed: choose deliberately between local, centralized, and reconciled — and define fail-open/fail-closed behavior.
- **429 ≠ 503.** Publish limits, emit remaining-budget headers, send `Retry-After`, and jitter everything.
- **Retry budgets, not retry counts.** Retry at one layer only.
- Measure to set limits; ship in observe-only mode first; alert on sustained per-tenant rejection.
