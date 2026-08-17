# System Design Prep

Study pack for platform/system-design rounds. One detailed writeup per question. Read this README first: it gives you a **repeatable structure** to drive every answer and a **pattern cheat sheet** for the concepts that recur across the questions.

## Files

### Agoda platform round (travel / booking theme)

| # | Question | File |
|---|----------|------|
| 1 | File storage service (S3 / Dropbox style) | [01-file-storage-service.md](./01-file-storage-service.md) |
| 2 | Flight aggregation / search system | [02-flight-aggregation-system.md](./02-flight-aggregation-system.md) |
| 3 | Scaling & reliability of the aggregator feed (rate limiting, API correctness, horizontal scaling, DB replication, testing, canary) | [03-aggregator-scaling-and-reliability.md](./03-aggregator-scaling-and-reliability.md) |
| 4 | Flight booking system (contention, payments, failure handling, aggregator sync) | [04-flight-booking-system.md](./04-flight-booking-system.md) |
| 5 | Event ticket booking (concurrency, no double-book, no oversell, abuse protection, resilience) | [05-event-ticket-booking-system.md](./05-event-ticket-booking-system.md) |

### Okta round (identity / infrastructure theme)

| # | Question | File |
|---|----------|------|
| 6 | Okta system design topics — speakable crib sheet across the recurring themes | [06-okta-system-design-topics.md](./06-okta-system-design-topics.md) |
| 7 | Backup & recovery system (consistency, RPO/RTO, restore guarantees, cost/scale tradeoffs) | [07-backup-and-recovery-system.md](./07-backup-and-recovery-system.md) |
| 8 | Push notification service (Kafka fan-out, provider abstractions, rate limiting, scheduling, dedup, retries) | [08-push-notification-service.md](./08-push-notification-service.md) |
| 9 | Single sign-on (SSO) system (token handling, session lifecycle, third-party IdP federation) | [09-signle-sign-on-system-design.md](./09-signle-sign-on-system-design.md) |

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

## Cross-cutting pattern cheat sheet — booking (3, 4, 5)

Know them cold; you'll reuse the same 6 patterns for three questions.

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

## Cross-cutting pattern cheat sheet — identity & infra (6, 7, 8, 9)

The same handful of ideas carry the Okta-flavoured questions.

### 1. Token vs session (the SSO fork in the road)
Stateful sessions (server-side store, easy revocation, needs a fast shared cache) vs stateless JWTs (no lookup, scales horizontally, but **revocation is the hard part**). The standard answer: short-lived access tokens + long-lived refresh tokens, plus a denylist/`jti` check or key rotation for emergency revocation. Say the tradeoff out loud — that's the graded part.

### 2. Federation protocols — SAML vs OIDC
SAML 2.0 for enterprise/browser SSO (XML assertions, POST binding); OIDC on top of OAuth2 for modern apps, SPAs, mobile, APIs. Core mental model is the same: the IdP holds *the* session, each SP gets its own derived session via a redirect + signed assertion/token. "Single" means one authentication at the IdP, not one cookie everywhere.

### 3. Single Logout, and third-party cookie deprecation
SLO is genuinely hard — front-channel iframes are fragile, back-channel needs every SP to implement a logout endpoint. Third-party cookie deprecation breaks the classic cross-domain tricks, which is a sharp, current thing to raise unprompted.

### 4. Keeping the IdP off the critical path
An IdP is a hard availability dependency for every downstream app. Cache/validate locally (public keys via JWKS, short token TTLs), multi-region active-active, and degrade gracefully rather than failing every login globally.

### 5. RPO / RTO drives every backup decision
Requirements first: how much data can you lose (RPO), how long can you be down (RTO). Everything else — full vs incremental vs differential, snapshot consistency, storage tiering, hot/warm/cold DR — falls out of those two numbers and the cost curve. **Untested backups don't exist**: restore drills are the answer to "where do real systems fail?"

### 6. Fan-out pipeline (notifications)
Ingest → partitioned queue → channel-specific dispatchers behind provider abstractions, with rate limiting, scheduling, dedup, and retries as *separate* concerns off the main path. Multi-tenancy means per-tenant quotas and noisy-neighbour isolation. Provider abstraction = swap vendors, per-channel retry/DLQ semantics.

Note the overlap with the booking patterns: idempotency (#3), async decoupling + DLQs (#6), rate limiting (#5), and the reliability building blocks above apply verbatim to the notification service.

---

## Interview-day tips
- **Lead with the summary, then the bullets.** State the one-line approach, then expand. Don't build the whole thing silently.
- **Drive the whiteboard.** Propose structure; let them redirect. Silence reads as being stuck.
- **Know the company's theme.** Agoda is **travel/booking** — expect pushes on **provider/GDS integration, availability freshness, price accuracy, overbooking, and payment failure handling**. Okta is **identity infrastructure** — expect **session/token lifecycle, SAML vs OIDC, single logout, revocation, multi-tenancy, and IdP availability**. Those are the "hard parts" each cares about operationally.
- When you don't know, **reason from first principles out loud** and state your assumption. That's the signal they're grading.
