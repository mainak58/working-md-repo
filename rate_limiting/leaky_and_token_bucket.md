# Rate Limiting Algorithms — A Plain English Guide

Rate limiting means controlling **how many requests someone can make, and how fast**. This guide covers the three most common algorithms and, more importantly, how they actually differ from each other.

---

## The one question that separates them all

> **When does my allowance come back?**

That's it. Every algorithm below is just a different answer to that question.

| Algorithm | Allowance comes back... |
|---|---|
| Concurrency limiter | when the previous request **finishes its work** |
| Leaky bucket | on a **clock**, at a steady drip, never accumulates |
| Fixed window | on a **clock tick**, all at once in one lump |
| Token bucket | on a **clock**, steady drip, **and unused amounts pile up** |

---

## 0. The common mistake: concurrency limiter

Before the real algorithms, here's the thing people *think* is a leaky bucket but isn't.

The pattern looks like this: you keep a counter starting at 10. A request arrives, you subtract one. Then you do the actual work — database queries, whatever it is. When that work finishes, you add one back to the counter before returning the response.

The giveaway is that last step: **the counter is restored on completion.**

This limits **how many requests run at the same time** — not the rate.

Why it's not rate limiting: if each request takes 1ms you'll serve thousands per second. If each takes 10 seconds you'll serve one per second. **Your limit is decided by how slow your database is, not by a number you chose.**

This is a semaphore. It's useful — it protects you from overload — but it is not a rate limiter.

> **Rule of thumb:** if your counter increments when work *completes*, it's a concurrency limiter. If it increments on a *timer*, it's a rate limiter.

---

## 1. Leaky bucket

**Mental model:** a waiting room with a door that opens on a schedule.

- The bucket is a **queue** with a fixed capacity (say 100)
- Requests arrive at any speed and line up in the queue
- They are released at a **constant rate** (say 10/second), driven by a timer
- If the queue is full, new arrivals are **dropped**

### How it plays out

Capacity 100, drain rate 10/sec. 100 requests arrive at 12:00:00.

```
12:00:00   100 requests arrive → all queued, queue is now full
12:00:01   releases 10   (90 left waiting)
12:00:02   releases 10   (80 left waiting)
12:00:03   releases 10   (70 left waiting)
...
12:00:10   releases the last 10
```

Any request arriving while the queue holds 100 → **dropped immediately.**

### The critical detail

The release schedule **does not care whether previous requests finished.**

If the request released at 12:00:01 takes 30 seconds to complete, the next batch still goes out at 12:00:02. Release and completion are completely decoupled. This is *not* `await` behaviour.

### Key property

**Output is perfectly smooth. Bursts are impossible.** Idle time earns you nothing — sit quiet for an hour and you still only get 10/sec when you come back.

---

## 2. Fixed window counter

**Mental model:** salary day. Your whole allowance lands at once.

- Pick a window (1 minute) and a limit (10 requests)
- Count requests in the current window
- When the clock ticks over to the next window, the counter **resets to full**

### How it plays out

```
12:00:00   counter = 10
12:00:00   user fires 10 requests → all pass, counter = 0
12:00:01   request → REJECTED (counter is 0)
12:00:30   request → REJECTED
12:00:59   request → REJECTED
12:01:00   counter jumps back to 10  ← lump refill
12:01:00   user fires 10 more → all pass
```

Note it's not "one request at a time while you wait." It's **fully blocked** until the reset, then **fully restored**.

### The boundary bug

This is the reason people avoid it:

```
12:00:59   10 requests  ✅ (legal — window 12:00)
12:01:01   10 requests  ✅ (legal — window 12:01)
```

**20 requests in 2 seconds** on a "10 per minute" limit. Both windows are individually valid, but the burst straddles the boundary. A fixed window can let through **double** your intended limit.

### Key property

Dead simple to implement, but spiky and exploitable at window edges.

---

## 3. Token bucket

**Mental model:** a wallet with automatic income.

- A bucket holds up to `N` tokens (say 100) — this is the **ceiling**, not a per-second figure
- A timer adds `R` tokens per second (say 10), never exceeding the ceiling
- Each request **spends one token**
- No token available → **rejected immediately.** There is no queue

### Two things that trip people up

**1. Tokens are not requests.** A token is a permission slip. It sits in the bucket waiting to be spent.

**2. The request never returns its token.** Only the timer refills. (Contrast with the concurrency limiter at the top.)

### How it plays out

Capacity 100, refill 10/sec. Bucket starts full after an idle period.

```
12:00:00   bucket = 100
12:00:00   user fires 100 requests → all 100 pass INSTANTLY, bucket = 0
12:00:00   the 101st request → REJECTED (no queue, no waiting)
12:00:01   +10 tokens → bucket = 10, user can make 10 requests
12:00:02   user makes only 8 → bucket = 2 leftover
12:00:03   +10 tokens → bucket = 12  ← leftovers carried forward
12:00:03   user can now make 12 requests
```

That carry-forward is the whole point. **Unused capacity accumulates** instead of being thrown away.

### Key property

Bursts are allowed *up to the bucket size*, but the long-run average is still capped at the refill rate. This matches how real traffic behaves — a page loads, fires 8 API calls at once, then goes quiet for 30 seconds.

---

## Side-by-side

| | Leaky bucket | Fixed window | Token bucket |
|---|---|---|---|
| **What's stored** | Requests (queued) | A count | Tokens |
| **Refill style** | Steady drip | One lump per window | Steady drip |
| **Idle time** | Earns nothing | Earns nothing | **Banks credit** |
| **Bursts allowed** | Never | Yes, at boundaries (bug) | Yes, by design |
| **Over-limit request** | Queued, or dropped if full | Rejected | Rejected |
| **Output rate** | Perfectly constant | Spiky | Variable, average-capped |

---

## Token bucket vs fixed window — the deciding test

These two get confused constantly. Run this test at **12:00:59**, after spending your full allowance early in the minute:

- **Fixed window:** you get **nothing** at 12:00:59, then **all 10 at once** at 12:01:00
- **Token bucket:** you get **1 token** at 12:00:59, **1 more** at 12:01:00, and so on

**Lump refill vs trickle refill.** That trickle is exactly why token bucket has no boundary bug — there's no magic instant where everything resets, so there's nothing to exploit.

---

## Which one should I use?

**Token bucket** — the default choice for APIs. It's what Stripe, AWS, GitHub, and Nginx's `limit_req burst=` use. Real users are bursty and that's fine, as long as the average holds.

**Leaky bucket** — when the thing downstream genuinely cannot handle spikes. Video streaming, network egress shaping, feeding a third-party API with a hard per-second cap, or protecting a fragile legacy service.

**Fixed window** — when you need something trivially simple and the boundary bug doesn't matter to you. Otherwise reach for a sliding window instead.

**Concurrency limiter** — not a rate limiter, but pair it *with* one. Rate limiter controls how fast requests arrive; concurrency limiter controls how many are in flight at once. They solve different problems.

---

## A note on implementation

You don't actually need a background timer running to refill buckets. The standard approach is to store a timestamp of the last refill, and whenever a request comes in, work out how much time has passed and add the tokens earned in that gap — capped at the bucket ceiling.

That capping step is what stops an idle user from banking unlimited credit. It's cheap, and it survives restarts, since everything is derived from timestamps rather than held in a running loop.

---

## Glossary

**Burst** — requests arriving in a sudden clump rather than evenly spread. Normal and expected in real traffic.

**Burst allowance** — in token bucket, the bucket capacity. The maximum number of requests that can fire at one instant.

**Sustained rate** — the refill rate. What you're limited to over the long run.

**Capacity (token bucket)** — a ceiling on the bucket, *not* a per-second quota.

**Capacity (leaky bucket)** — the queue size. How many can wait before you start dropping.
