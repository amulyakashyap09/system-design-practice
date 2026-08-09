# 3. Scaling & Reliability of the Aggregator Feed

> **Setup:** data supplied to multiple airline aggregators. Issues discussed: no rate limiting, incorrect API methods, lack of horizontal scaling, no DB replication. Also: software testing, rolling out to limited traffic. (This was a ~40-min discussion, so they want *depth per issue*, not a fresh design.)

**How to frame it:** This is not a greenfield design question — it's "here's a system with concrete production problems, diagnose and fix each, then talk about how you'd safely ship the fixes." Treat each bullet as its own mini deep-dive with: *why it's a problem → the fix → the tradeoff → how you'd verify it.* That structure is the whole game here.

**One-liner:** Make the service **stateless and horizontally scalable** behind a load balancer, add **distributed rate limiting** (token bucket) both inbound and toward providers, **correct the API semantics** (right HTTP methods, idempotency, proper status codes), add **DB replication** (primary + read replicas with failover), and ship all of it safely through **layered testing + canary/progressive rollout**.

---

## Issue 1 — No rate limiting

**Why it's a problem**
- **Inbound:** any client (or a bug/bot) can hammer you → resource exhaustion, noisy-neighbor, cascading failure. One misbehaving consumer degrades everyone.
- **Outbound:** you feed *multiple aggregators/providers*; without limiting you exceed **their** quotas → they throttle or ban you, and your fan-out (see Q2) amplifies everything.

**Fix**
- **Token bucket** as the default algorithm: capacity = burst allowance, refill rate = sustained rate. Allows short bursts, caps sustained load. (Sliding-window-log if you need strict fairness; fixed-window-counter is simplest but has boundary-burst issues.)
- Enforce **per API key / per client / per IP**, and separately **per provider** on the outbound side.
- Make it **distributed** — because you're about to scale horizontally, the limit must be shared across all instances. Implement with **Redis** (`INCR`+`EXPIRE`, or a Lua token-bucket script for atomic check-and-decrement). A per-instance in-memory limiter is wrong once you have N instances (effective limit becomes N×).
- Respond `429 Too Many Requests` with `Retry-After`. Optionally tiered limits per client plan.

**Tradeoff:** central Redis limiter adds a network hop + a dependency; mitigate with local pre-check + short-lived local token leases, and fail-open vs fail-closed decision per endpoint.

**Verify:** load test that a client over quota gets 429s and others are unaffected; confirm outbound calls stay under each provider's ceiling.

## Issue 2 — Incorrect API methods

**Why it's a problem**
- Using wrong HTTP semantics breaks correctness and safety: e.g. a **non-idempotent GET** (GET with side effects) gets retried/cached by proxies and CDNs and silently duplicates actions; using `POST` where `PUT`/idempotency is needed means **retries create duplicates** (critical when this feeds bookings/payments); wrong status codes make clients mis-handle failures (treating a retriable 503 as a permanent error, or a 200 that was actually a failure).

**Fix**
- Apply REST semantics correctly: **GET = safe + idempotent, no side effects; PUT/DELETE = idempotent; POST = create/non-idempotent**. Cacheable reads = GET.
- Add **idempotency keys** to all mutating calls (see cheat sheet #3) so client/network retries don't double-execute — mandatory when talking to airline/payment systems where a timeout doesn't tell you if the other side committed.
- Correct **status codes** and error contracts: `4xx` client error (don't retry), `429`/`503` retriable (retry with backoff+jitter), `5xx` server error. Consistent error schema so callers can react programmatically.
- Correct **content negotiation, pagination, and versioning** (`/v1/`), so clients don't break on change.

**Tradeoff:** enforcing idempotency needs a store of seen keys (TTL'd in Redis) — small cost for large safety gain.

**Verify:** contract tests / schema validation in CI; a retry test that proves duplicate submits are collapsed.

## Issue 3 — Lack of horizontal scaling

**Why it's a problem**
- A single (or vertically-scaled) instance is a hard ceiling and a **single point of failure**. Under peak (holidays, marketing) it saturates; one crash = outage.

**Fix**
- Make app servers **stateless**: no in-memory session/affinity. Push session/state to **Redis** or the DB, so any instance serves any request. Statelessness is the *precondition* for horizontal scale.
- Put instances behind a **load balancer** (L7), health-checked, and **auto-scale** on CPU/QPS/latency. Add nodes to absorb load; remove when idle.
- For the data tier, scale reads with **replicas** (Issue 4) and, if writes are the ceiling, **shard**/partition by a natural key (e.g. route or provider) with **consistent hashing** to minimize reshuffling when nodes change.
- Decouple slow/spiky work onto a **queue** (Kafka/SQS) with independently-scaled consumers → smooths spikes, isolates failures (backpressure).

**Tradeoff:** statelessness sometimes means an extra Redis lookup; sharding adds cross-shard query complexity. Both are worth it for elasticity + fault tolerance.

**Verify:** load test showing near-linear throughput as you add instances; kill an instance and confirm no user-visible impact.

## Issue 4 — No DB replication

**Why it's a problem**
- Single DB = **SPOF** (its failure = total outage + potential data loss) and a **read bottleneck** (all reads + writes hit one node).

**Fix**
- **Primary–replica replication:** primary takes writes; **read replicas** serve read traffic (this system is read-heavy — aggregation/search). Route reads to replicas, writes to primary.
- **Automatic failover:** promote a replica if the primary dies (managed via the DB's HA tooling / an orchestrator), with health checks and a floating endpoint so apps reconnect transparently.
- Choose replication mode by durability need: **synchronous/semi-sync** for zero-data-loss on the primary (higher write latency) vs **async** for lower latency (small data-loss window on failover). State the tradeoff explicitly.
- Cross-AZ/region replicas for DR and read locality.

**Watch out for replication lag → read-your-writes:** a user who just wrote may read a stale replica. Fixes: route that user's immediate reads to the primary, or use a write-through cache, or "read from primary for N seconds after a write."

**Tradeoff:** replicas cost money and add lag; sync replication costs write latency. Pick per consistency requirement.

**Verify:** failover drill (kill primary, measure recovery time + data loss); confirm read traffic offloads to replicas.

## Other topics they raised

### Software testing (how you'd assure these fixes)
- **Unit tests** — logic (rate-limiter math, idempotency dedup, ranking).
- **Integration/contract tests** — provider adapters against mocked provider APIs; verify correct methods/schemas (catches Issue 2 regressions).
- **Load/stress tests** — prove horizontal scaling and rate-limit behavior under peak.
- **Chaos/fault-injection** — kill a provider, kill the primary DB, inject latency → validate circuit breakers, failover, graceful degradation.
- **Regression + CI gates** — no deploy without green.
- **Canary/synthetic monitoring** in prod — continuous probes that catch issues before users do.

### Rolling out to limited traffic (safe deploys)
- **Canary release:** ship to a small % of traffic/instances first, watch RED metrics (Rate, Errors, Duration) + business KPIs; **auto-rollback** on regression. Progressively ramp 1% → 5% → 25% → 100%.
- **Feature flags:** decouple deploy from release; turn features on for a cohort, kill instantly if bad — no redeploy.
- **Blue-green:** run new version alongside old, flip the LB, instant rollback by flipping back.
- **Shadow/dark traffic:** mirror real traffic to the new version without serving its responses — validate under real load risk-free.
- Pair with **observability + alerting** so the canary's health is measurable and rollback is automatic.

*(You've done canary/Spinnaker/observability work — say so; this is your home turf.)*

---

## Closing synthesis (say this to tie it together)
These aren't four unrelated bugs — they compound. **Statelessness (3)** enables horizontal scale, which *forces* the rate limiter to be **distributed (1)**; horizontal scale is only fault-tolerant if the **DB is replicated (4)**; and **correct, idempotent API semantics (2)** are what make retries safe across all those extra instances and failovers. You ship the whole thing behind **canary + flags** so a mistake in any of it is caught at 1% of traffic, not 100%.
