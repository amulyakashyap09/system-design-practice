# Agoda Platform Round — System Design Prep

Study pack for the platform/system-design round. Five questions, one detailed writeup each. Read this README first: it gives you a **repeatable structure** to drive every answer and a **pattern cheat sheet** for the concepts that recur across the flight/booking/ticketing questions.

## Files

| # | Question | File |
|---|----------|------|
| 1 | File storage service (S3 / Dropbox style) | `01-file-storage-service.md` |
| 2 | Flight aggregation / search system | `02-flight-aggregation-system.md` |
| 3 | Scaling & reliability of the aggregator feed (rate limiting, API correctness, horizontal scaling, DB replication, testing, canary) | `03-aggregator-scaling-and-reliability.md` |
| 4 | Flight booking system (contention, payments, failure handling, aggregator sync) | `04-flight-booking-system.md` |
| 5 | Event ticket booking (concurrency, no double-book, no oversell, abuse protection, resilience) | `05-event-ticket-booking-system.md` |

## How to drive any system design answer (say this structure out loud)

1. **Clarify + scope (2–3 min).** Functional requirements, then non-functional (scale, latency, consistency, availability). State what's out of scope. Interviewers reward you for narrowing before designing.
2. **Back-of-envelope numbers.** QPS (read vs write), storage/day, bandwidth, peak-to-average ratio. Numbers justify every later choice (cache size, shard count, replica count).
3. **API design.** A handful of endpoints. Call out idempotency and pagination.
4. **Data model + storage choice.** Pick SQL vs NoSQL vs object store vs cache *and say why* (access pattern, consistency need).
5. **High-level architecture.** LB → stateless API tier → services → data tier → async workers → cache/CDN. Draw the boxes.
6. **Deep dives** on whatever the interviewer pushes — this round *is* the deep dives. The scraped questions tell you exactly where they push: concurrency, overselling, payments, provider failures.
7. **Bottlenecks, tradeoffs, "what I'd do next."** Name what breaks first and how you'd evolve it. Senior signal = knowing where the design is weakest.

Golden rule: **there is no single right answer; there is defended reasoning.** Always frame as "I'd use X because <access pattern / consistency / failure model>, trading off Y."

---

## Cross-cutting pattern cheat sheet

These appear in 3, 4, and 5. Know them cold; you'll reuse the same 6 patterns for three questions.

### 1. Reservation / hold with TTL (the anti-double-booking primitive)
Don't hold a DB row lock across the seconds/minutes a human takes to pay. Instead:
- Create a short-lived **reservation** (seat/room/ticket) with an expiry (e.g. 5–10 min).
- Confirm on payment success; a **sweeper** (or lazy expiry) frees abandoned holds.
- Backing store options: a row with `status + expires_at`, or Redis key with TTL. Redis gives you atomic ops + auto-expiry; the SQL row gives you durability. Often: Redis for the fast hold, SQL as source of truth on confirm.

### 2. Preventing oversell — make the decrement atomic
The bug is always **read-then-write race**. Fixes, cheapest first:
- **Conditional update in the DB:** `UPDATE inventory SET available = available - 1 WHERE id = ? AND available > 0` — the WHERE guard + single-statement atomicity means only one winner. Check `rowsAffected`.
- **Optimistic locking:** version column, `WHERE version = ?`, retry on conflict. Good for low contention.
- **Pessimistic locking:** `SELECT ... FOR UPDATE`. Correct but serializes; only for genuinely hot rows.
- **Redis atomic counter / Lua script:** `DECR` guarded, or a Lua script for check-and-set atomicity. Fast, but Redis is now on your correctness path → needs durability/reconciliation.
- **Unique constraint as a backstop:** `UNIQUE(event_id, seat_id)` on the bookings table means the DB itself refuses the second insert even if app logic races.

### 3. Idempotency (mandatory anywhere money or external calls happen)
Client sends an **idempotency key** per logical operation. Server stores `key → result`; a retry with the same key returns the stored result instead of re-executing. Essential for: payment submit, booking confirm, and any call to an airline/provider where a network timeout doesn't tell you whether the other side committed.

### 4. Saga for multi-step distributed transactions
No 2PC across payment provider + inventory + aggregator. Use a **saga**: a sequence of local transactions, each with a **compensating action**. Booking saga: `reserve → charge → confirm → notify`; if `confirm` fails, compensate by `refund + release`. Drive it with an orchestrator (explicit state machine) or choreography (events). Pair with the **transactional outbox** so state change + event emission are atomic. (You already know this vocabulary — use it.)

### 5. Rate limiting
- **Token bucket** (allows bursts, refills at fixed rate) is the default answer. Sliding-window-log for precise fairness; fixed-window-counter is simplest but has boundary bursts.
- Enforce **per-client / per-API-key / per-IP**, distributed via Redis (INCR + EXPIRE, or a Lua token-bucket).
- Return `429` + `Retry-After`. Protects *you* from clients and, on the outbound side, keeps you under a provider's quota.

### 6. Async decoupling + backpressure
Anything slow, spiky, or best-effort goes on a **queue** (Kafka/SQS): payments confirmation emails, aggregator sync, search index updates. Buffers spikes, isolates failures, enables retries with **dead-letter queues**. Consumers must be idempotent (see #3).

### Reliability building blocks referenced throughout
- **Horizontal scaling** needs **stateless** app servers (session/state pushed to Redis/DB) so any instance serves any request and you can add nodes behind the LB.
- **DB replication:** primary for writes, read replicas for read scale + failover; understand **replication lag** and read-your-writes issues.
- **Circuit breaker + timeout + retry-with-jitter + bulkhead** around every external dependency (provider APIs, payment gateway).
- **Caching layers:** CDN (static/edge), Redis (hot data), with explicit TTL and invalidation strategy.
- **Observability:** RED metrics (Rate, Errors, Duration), tracing across the saga, alerting.

---

## Interview-day tips
- **Lead with the summary, then the bullets.** State the one-line approach, then expand. Don't build the whole thing silently.
- **Drive the whiteboard.** Propose structure; let them redirect. Silence reads as being stuck.
- Agoda is a **travel/booking** company — every question here is on-theme. Expect them to push on **provider/GDS integration, availability freshness, price accuracy, overbooking, and payment failure handling**. Those are the "hard parts" they care about operationally.
- When you don't know, **reason from first principles out loud** and state your assumption. That's the signal they're grading.
