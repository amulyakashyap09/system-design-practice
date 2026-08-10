# 2. Flight Aggregation System (flight search across many providers)

**One-liner:** A search service that fans out a query to many airline/GDS/aggregator providers in parallel with per-provider timeouts and circuit breakers, normalizes and merges their responses, caches results with short freshness-aware TTLs, and returns ranked results — degrading gracefully to partial results when some providers are slow or down.

This is the classic Agoda-shaped problem: **you don't own the inventory; you integrate with many external providers whose data is slow, rate-limited, and constantly changing.** The design is dominated by that reality.

---

## 0. Frame the problem before designing anything

Say this out loud in the first 60 seconds:

> "I don't own the inventory. My data lives behind ~20 third-party APIs that are slow (2–10s), rate-limited, unreliable, and each returns a different schema. So this isn't a storage or sharding problem — it's a **dependency management** problem."

**Why this framing matters more than it looks:** it eliminates the entire standard toolbox. Normally your levers are indexing, sharding, replicas, denormalization — all of which assume *you control the data store*. Here you control none of it. You cannot add an index to Amadeus. You cannot shard Emirates. The only levers left are:

1. **Caching** — avoid the call entirely
2. **Concurrency & fan-out control** — parallel, but bounded
3. **Timeouts and deadlines** — bound the damage of a slow call
4. **Graceful degradation** — a missing provider is a degraded result, not an error
5. **Normalization** — make heterogeneous things comparable

Everything in sections 1–7 is one of those five. Name the constraint first and the rest of the design reads as inevitable rather than as a list of buzzwords.

**Second framing point — separate search from booking.** Search is read-heavy, high-volume, and *tolerant of approximation*: a price 3 minutes old is fine. Booking is a low-volume write that must be exact and transactional. Conflating them is the most common failure here — you end up demanding real-time accuracy on a path taking 12,000 req/s, and the design becomes impossible. **This system is search-only.** Booking is a separate system (see `04-flight-booking-system.md`).

---

## 1. Clarify requirements

**Functional**
- Search flights by origin, destination, dates, passengers, cabin.
- Aggregate from N providers (airline APIs, GDS like Amadeus/Sabre, other aggregators).
- Normalize heterogeneous responses into one schema.
- Rank/sort (price, duration, stops, provider reliability).
- Filter (airlines, times, stops, baggage).
- Return latest **available price + availability** (this must be reasonably fresh — stale price → failed booking later).

**Non-functional**
- **Latency:** search must feel fast (<1–2s p95) even though some providers take 5–10s.
- **Availability:** one provider down must not break search.
- **Freshness vs speed tradeoff** is the central tension.
- High read volume, spiky (marketing, holidays).

**Why the functional list gets 30 seconds and the non-functional gets 5 minutes:** functionally this is a `for` loop over HTTP clients and a `sort()`. A junior can build it. The entire difficulty is *"how is it fast and available when the dependencies are neither?"* Dwelling on functional scope signals you haven't found where the difficulty is.

### The four clarifying questions — and why each is load-bearing

Every one of these changes the architecture, which is why you ask rather than assume.

| Question | What it changes |
|---|---|
| **How many providers, what SLAs/quotas?** | Sets fan-out width (20 vs 200 is a different system), sets your latency floor (you can't beat your fastest provider on a miss), determines whether you need bounded concurrency and per-provider bulkheads. |
| **How fresh must prices be?** | Sets TTL → sets cache hit rate → sets outbound call volume → determines whether you fit inside provider quotas *at all*. This one answer cascades through the entire design. Highest-leverage question in the interview. |
| **Search-only, or does it feed booking?** | If it feeds booking, cached prices need a re-validation step before commit, and you must track price-change-at-booking. Pure metasearch can be far more aggressive with staleness. |
| **Push or pull?** | If any provider streams updates, you can keep a warm cache for their inventory and **eliminate per-search fan-out for them entirely**. Not a tuning change — a different architecture. |

**The tension to name explicitly:** fresh data requires calling the provider (slow, burns quota); fast responses require serving cache (risks staleness, causes failed bookings downstream). **Every subsequent decision is a position on that one axis.** Saying this early gives you a consistent story to hang sections 4, 6 and 7 on.

---

## 2. Back-of-envelope

- 10M searches/day ÷ 86,400 ≈ **~120 searches/sec** average.
- Peak multiplier 3–5× (holidays, marketing, morning browsing) → **~600/sec peak**.
- Fan-out to 20 providers with zero cache → **12,000 outbound provider calls/sec at peak.**

**Why this is the most important calculation in the design:** no airline or GDS will give you 12,000 rps. Real quotas are 10–100 rps, often with contractual *look-to-book ratio* penalties on top. You are **100–1000× over budget**. That's not a performance problem to tune later — the naive design is *impossible*, not merely slow.

Then invert it to get the actual requirement:

> If a provider allows 50 rps and you'd otherwise send 600/s, you need **~92% cache hit rate on that provider** just to be legal.

Now "we'll add a cache" stops being a hand-wave and becomes a numeric target. That's what makes section 6b feel earned rather than reflexive.

### Latency math

- 20 providers sequentially × ~2s = **40s**. Dead on arrival → **parallel fan-out is mandatory**, not an optimization.
- Parallel means **total latency = max(provider latencies)** — you inherit the *tail* of your worst dependency.
- With 20 providers, if each has a 5% chance of being slow, P(at least one slow) = 1 − 0.95²⁰ ≈ **64%**.

**Why that number matters:** the slow case isn't an edge case, it's the *majority* case. That's the mathematical justification for progressive results (§3) and deadlines (§6a). Waiting for everyone means being slow ~64% of the time.

### Storage sizing

~50 results × ~1KB = ~50KB per `(route, date, provider)`. Hot working set — top ~1,000 routes × 30 departure dates × a few pax/cabin combos × 20 providers — lands in the **tens of GB**.

**Why compute this:** it proves the cache fits in Redis and that you don't need a database on the hot path, which is exactly the conclusion §4 draws.

---

## 3. API design

```
POST /search
  body: { origin, destination, departDate, returnDate?, pax:{adt,chd,inf}, cabin }
  -> { searchId, results:[...], partial:bool, version, completedProviders, pendingProviders }

GET  /search/{searchId}?since={version}   # poll for late results (delta)
GET  /search/{searchId}/stream            # SSE: push results as each provider returns
```

**Why POST, not GET:** the query is a structured object — pax split into adults/children/infants, optional multi-city legs, cabin, filters. Awkward as a query string, and you cache internally on a normalized key anyway, so HTTP-layer cacheability buys nothing. *(Counter-argument worth acknowledging: GET would let a CDN cache popular routes at the edge — but personalization and the pax dimension fragment it badly.)*

**Why `searchId` exists at all:** it converts a search from *one blocking call* into a **stateful, resumable resource**. That's the enabling primitive for everything else — without an ID there's no way to hand back late results, and you're forced to block on the slowest provider. The partial result set lives in Redis keyed by `searchId`, which is what lets *any* stateless instance serve the follow-up poll (the poll almost certainly hits a different box than the POST did).

**Why `completedProviders` / `pendingProviders` are in the contract:** honesty. Without them the client cannot distinguish **"there are no cheap flights"** from **"we haven't finished looking."** That ambiguity is a real UX bug: users filter to "under $400", see nothing, and leave — while the provider holding a $380 fare was still in flight.

**Why progressive delivery is the central API decision:** it decouples *first paint* from *completeness*. Blocking makes p95 equal to the worst provider's tail. Progressive makes first-paint equal to cache/fastest-provider latency while total work stays identical — same backend cost, order-of-magnitude better perceived performance. This is how real metasearch feels snappy despite 8-second GDS calls.

### Mechanically: how a slow provider's results get appended

The enabling idea: **the search is a mutable resource in Redis, not an in-flight HTTP request.** Provider tasks are *detached from the request lifecycle*, so the HTTP response returns while they're still running.

**Redis state created by `POST /search`** (all TTL ~10 min):

```
search:{id}:meta       HASH   {query, createdAt, deadline, version}
search:{id}:results    ZSET   score = rank score, member = offer JSON / offer id
search:{id}:providers  HASH   {amadeus:"done", sabre:"pending", ba:"timeout", ...}
search:{id}:dedup      SET    canonical flight keys (append-time dedup)
search:{id}:events     PUBSUB channel
```

**Flow:**

1. POST creates `searchId`, writes meta, marks all providers `pending`, and dispatches N provider tasks to a worker pool **detached from the request** (not awaited by the handler).
2. Global deadline fires at ~1.5s → the handler reads whatever is in `search:{id}:results` and returns it with `partial:true`.
3. Provider F finishes at 6.2s. Its worker does:
   `normalize` → `SADD search:{id}:dedup <canonicalKey>` (returns 0 → duplicate → keep the cheaper) → `ZADD search:{id}:results <score> <offer>` → `HSET …:providers f done` → `HINCRBY …:meta version 1` → `PUBLISH search:{id}:events {version, added:[…]}`.
4. **It also writes the provider cache entry** `(route,date,pax,cabin,provider)` regardless of whether the user is still there.

**Why step 4 matters:** late work is never wasted — even if the user left, it warms the cache for the next one. Given §2's hit-rate requirement, that's free progress toward the constraint.

**Why Redis Pub/Sub is required:** the SSE connection is almost certainly on a *different app instance* than the one that ran the POST (stateless fleet behind a load balancer). The instance holding the SSE socket does `SUBSCRIBE search:{id}:events` on connect; the worker `PUBLISH`es. That's the cross-process bridge. Pub/Sub is fire-and-forget, which is safe **because the durable state is also in Redis** — a dropped notification is recovered by the client's next poll or reconnect.

**Transport, in preference order:**

| | Mechanism | Why |
|---|---|---|
| Primary | **SSE** `/search/{id}/stream` | Unidirectional server→client, plain HTTP, and crucially auto-reconnects with `Last-Event-ID` — the server replays only events after that version. Resumption is free. |
| Fallback | **Polling** `/search/{id}?since={version}` | For proxies/clients that kill long-lived connections. `since` makes it a cheap delta read (`ZRANGEBYSCORE` on what's new). |
| Only if needed | WebSocket | Earns its complexity only if the client must talk back mid-search ("stop, I picked one"). |

**Send deltas with scores, not re-ranked full lists.** Each event carries `{version, added:[offers with scores], providerStatus}`; the client inserts by score.

**Why:** re-sending everything means a 50-offer payload per provider completion (~20× the bandwidth) and, worse, lets the visible order shuffle under the user's cursor. Scored inserts mean existing rows never move — new rows slot in. **Stable ordering is a requirement, not a nicety.**

**Why `version` exists:** the client may receive the same event from both the stream and a reconnect poll. The version lets it discard what it already applied — idempotent delivery.

**Model provider states properly:** `pending | done | empty | timeout | failed | circuit_open | rate_limited`. Collapsing these to done/not-done is a mistake — the UI genuinely needs to distinguish "Sabre found nothing" from "Sabre never answered."

---

## 4. Data model / storage

Three things: adapter config, a Redis cache, one normalized result schema. Notably **no primary database on the hot path**.

### 4a. Cache key: `(origin, dest, date, pax, cabin, provider)`

**Why the key includes `provider` — i.e. why cache per-provider, not per-search.** This is the highest-value detail in this section. Caching the whole merged result under one key means:
- one provider missing invalidates the *entire* entry,
- zero reuse across searches that overlap partially,
- no way to give different providers different TTLs.

Per-provider keys give **partial hits**, which are the normal case: 14 of 20 providers hit cache, you fan out to 6. Effective fan-out drops from 20 → 6 with zero staleness cost on the 14.

> **The granularity of your cache key *is* your unit of reuse.** Here that's the difference between fitting in provider quota and not.

**Why keys must be canonicalized:** `LHR`/`lhr`, `2026-03-05`/`03/05/2026`, `{2 adults}`/`{adt:2,chd:0,inf:0}` are the same query. Different keys → fragmented cache → collapsed hit rate → (per §2) quota breach. So: uppercase IATA, ISO-8601 dates, canonically-ordered pax tuple, cabin enum, then hash for a compact key.

**Why store `fetched_at` alongside the value:** a TTL tells you the entry is *alive*, not how *old* it is. You need explicit age to (a) implement stale-while-revalidate, (b) display "prices as of 14:32", (c) decide at booking time whether the offer needs re-validation. Age is first-class data here.

### 4b. Normalized result schema

```
Flight {
  segments[], price{…}, fareClass, baggage, provider,
  deeplink / offerToken, fetchedAt
}
```

**Why `offerToken` / `deeplink` is critical:** the offer is a provider-specific opaque handle. At booking you **replay their token back to them** — you never reconstruct a booking from your normalized fields. This is what makes normalization *safe*: you can lose fidelity flattening 20 schemas into 1 (and you will) without ever losing the ability to actually book the thing. **Normalization is for comparison; the token is for transaction.**

### 4c. Config store for adapters — three tiers

The requirements conflict: read on every request (sub-microsecond), auditable and rollback-able (needs a real DB), kill switch must propagate in seconds (needs push). So:

```
Postgres  (source of truth: versioned, audited, admin-UI writable)
   │  on write → PUBLISH config:changed
   ▼
Redis     (distribution + fast kill-switch keys)
   │  pub/sub notify + 30s poll as backstop
   ▼
In-process hashmap in every app instance  ←── read on the hot path
```

**Why Postgres as source of truth:** row-level history (`provider_config_versions`), transactional admin writes, and `rollback to version N` when someone fat-fingers a timeout. *Git-backed YAML deployed by CI is an equally valid answer* and gives you code review on config changes — pick one and justify it. Postgres wins if non-engineers edit it; git wins if only engineers do.

**Why never read config over the network on the hot path:** it's read once per provider per request. 1ms × 20 providers × 600 rps means you've built a second, worse rate limiter. In-memory map, refreshed in the background.

**Why pub/sub *and* a 30s poll:** pub/sub is fire-and-forget — an instance starting up or briefly disconnected misses the message and serves stale config forever. **The poll is the convergence guarantee; pub/sub is the latency optimization.**

**Split refresh cadence by field volatility:**
- `enabled` / kill switch → Redis key, **2–5s** in-memory TTL. *Why:* disabling a provider during an incident must not wait 30s.
- timeouts, limits, breaker thresholds → 30s.
- schema mapping, endpoints → on notify only.

```yaml
provider_id: amadeus
enabled: true
endpoint: { base_url: "...", protocol: rest, search_path: /v2/shopping/flight-offers }
auth:     { type: oauth2, secret_ref: "vault://providers/amadeus" }   # reference only
limits:   { rps: 50, burst: 100, daily_quota: 2_000_000, max_concurrency: 20 }
timeouts: { connect_ms: 500, read_ms: 3000, total_ms: 3500 }
breaker:  { error_rate: 0.5, window_s: 30, min_calls: 20, cooldown_s: 60 }
cache:    { ttl_s: { far: 900, near: 120, last_minute: 30 } }
markets:  [SEA, EU]
reliability_score: 0.972      # written by an offline job, never by hand
```

**Why the secret is a reference, not a value:** config is readable by anyone with admin-UI access and ends up in logs and debug endpoints. Credentials need separate ACLs and independent rotation. The adapter resolves `secret_ref` at startup and on a rotation TTL.

**Why `reliability_score` lives here but is machine-written:** ranking (§6e) needs it on the hot path, so it belongs with the config — but it's derived from booking outcomes, so a nightly job owns that column. Humans must not edit it.

**Two things people miss:**
- **Validate and canary config changes.** A bad config change is as dangerous as a bad deploy but bypasses every deploy safety rail. Schema-validate on write; roll to one instance and watch its error rate before the fleet.
- **Complex response mapping belongs in code, not config.** A JSON-transform DSL in the DB is unmaintainable and untestable the moment a provider nests fares three levels deep. **Config holds knobs; the adapter class holds parsing.**

### 4d. Why no primary DB on the hot path

You have no owned source of truth — it lives at the providers. A database here would be a stale copy of someone else's data plus an extra failure mode and network hop. So the storage tier is **Redis (cache) + config store + logs**. Raw provider responses ship asynchronously to object storage — you need them to debug mapping bugs and, more usefully, to **re-normalize historical data** after fixing a schema mapping. Analytics and price history go offline to a columnar store, deliberately off the serving path.

---

## 5. High-level architecture

```
Client
  │  POST /search
  ▼
API / Search Orchestrator (stateless)
  ├─ 1. normalize query, build cache key
  ├─ 2. cache lookup (Redis) — return hits immediately
  ├─ 3. fan-out to Provider Adapter layer for misses (parallel, bounded)
  │        ├─ Adapter A ─(breaker → limiter → bulkhead → timeout)→ Provider A
  │        ├─ Adapter B ─ ... → Provider B
  │        └─ Adapter N ─ ... → Provider N
  ├─ 4. normalize + merge + dedup + rank
  ├─ 5. write results back to cache with TTL
  └─ 6. return partial now, stream the rest
```

The order is deliberate:

**1. Normalize the query and build the key — first.** Every downstream decision keys off it; normalizing after the lookup would defeat the lookup.

**2. Cache lookup — before fan-out.** It's the cheapest path, and more importantly **its result determines what you actually need to fan out to.** You're not choosing between cache and fan-out; you're using the cache to *shrink* the fan-out.

**3. Fan out to misses, parallel and bounded.**
- *Why parallel:* §2's arithmetic (sequential = 40s).
- *Why bounded:* unbounded fan-out under a spike simultaneously (a) blows provider quotas and effectively DDoSes a partner and (b) exhausts your own connection pools and worker threads. Bound with a per-provider semaphore — a **bulkhead** — so one pathologically slow provider can't consume every connection in the shared pool. That failure ("one slow dependency saturates the pool and takes down *all* requests, including ones that don't touch it") is the classic cascading failure; bulkheads are the direct fix.

**Why the adapter abstraction exists.** Providers differ in auth (OAuth2 / static key / mTLS / SOAP WS-Security), protocol (REST-JSON vs SOAP-XML vs a stateful GDS session you must open, use and close), pagination, error semantics, rate limits, and schema. If any of that leaks into the orchestrator, the core becomes a tangle of `if (provider === 'sabre')`. The adapter contract is one function:

```
search(NormalizedQuery) → NormalizedResult[]     # + its own limiter, timeout, breaker
```

**Why the interviewer cares:** onboarding provider #21 should be *a new adapter plus a config row*, with zero core changes. That's open/closed — and here it isn't academic: adding providers is the literal business roadmap. A design where growth requires editing the orchestrator is a design that doesn't survive the company's success.

**4–6. Normalize → merge/dedup/rank → write cache → return partial.**

**Why cache the *normalized* form, not the raw response:** a cache hit should cost near-zero CPU. Caching raw XML makes every hit pay parsing and transformation — and at a 90%+ hit rate that becomes the dominant CPU cost in the whole system. **Cache the expensive-to-produce artifact, not the input.**

---

## 6. Deep dives

### 6a. Fan-out, timeouts, deadlines, breakers

**How:** `Promise.allSettled` semantics — collect whoever finishes, ignore the rest. Hard per-provider timeout (~3s) **plus** a separate global deadline (~1.5–2s).

**Why timeouts are non-negotiable:** without one, a hung provider socket holds a request slot indefinitely. Under sustained load slots fill, and a *single* misbehaving partner becomes a *total* outage. **The timeout is the mechanism that converts a dependency failure into a partial result instead of an outage.** That conversion is the whole game.

**Why a global deadline is separate from per-provider timeouts:** they answer different questions. The per-provider timeout says *"give up on this call."* The deadline says *"give up on waiting and respond to the user now."* If 19 providers return in 400ms and one is at 2.9s, per-provider timeouts alone still make everyone wait 2.9s. The deadline caps user-facing latency independently of any individual call's budget.

#### Where exactly the circuit breaker lives

**Inside the adapter, wrapping the outbound call, keyed per provider — local decisions, asynchronously shared state.** The order within the adapter is deliberate:

```
adapter.search(query):
  1. breaker.check(provider)          → OPEN?  return Unavailable immediately (0 cost)
  2. rateLimiter.tryAcquire(provider) → denied? return RateLimited (serve cache)
  3. bulkhead.acquire(provider)       → bounded concurrency semaphore
  4. httpCall(timeout)                → the actual request
  5. breaker.record(outcome)          → success | failure | timeout
  6. normalize()
```

**Why the breaker is step 1, before the rate limiter:** if the provider is dead you don't want to *spend a token* discovering that. Tokens are your scarcest resource. Fail fast at zero cost.

**Why per-provider, never global:** the entire purpose is blast-radius isolation. A global breaker means one sick provider silences all twenty. Go finer (`(provider, region)`) only if that's a real failure domain — over-partitioning means no key ever accumulates enough samples to trip.

**Why the decision must be local (in-memory), not in Redis:**
- it adds a round trip to the fast-fail path, which is supposed to be free;
- it creates a circular dependency — your resilience mechanism now fails when your cache fails.

**But pure-local has a real flaw:** with 50 instances each sees 1/50 of traffic, so on a low-QPS provider no single instance accumulates enough failures to trip — and each instance independently pays the full timeout learning the provider is dead, 50 times over.

**So: hybrid.** Local in-memory breaker makes every decision. Each instance async-publishes failure counters to Redis (`HINCRBY health:{provider} fail 1`, fire-and-forget, off the request path). A background loop reads the fleet aggregate every few seconds and can **force-open the local breaker**. Local trips fast on strong local signal; the aggregate catches slow-burn and low-QPS cases and makes the fleet converge.

**Trip on error *rate* over a rolling window, gated by minimum volume** — e.g. ≥20 calls in 30s **and** >50% failures. *Why the volume gate:* 1 failure out of 1 call is a 100% error rate and statistically meaningless; without it the breaker flaps constantly on low-traffic providers.

**What counts as a failure — the classic bug:** timeouts, connection errors and 5xx count. **`200 OK` with zero flights does not.** Neither does `400 invalid route`. Those are *correct* responses. Counting empty results as failures trips your breaker on perfectly healthy providers on obscure routes — a genuinely common production incident.

**Half-open:** allow *one* concurrent probe, not a flood. Releasing all traffic on half-open while the provider is still sick re-hammers it at the worst possible moment and it never recovers.

**When open, degrade in order:** stale cache → omit the provider and mark partial → surface `circuit_open` in the providers hash so the UI says "unavailable" rather than implying no inventory.

#### Retries — why to be stingy

These are idempotent reads, so retrying is *safe*, but **retry amplification** is the danger: 20 providers × 3 retries = 60 calls per search, and retries arrive precisely when a provider is already browning out. That's the mechanism that converts a brownout into a full outage. So: at most one retry, exponential backoff with jitter, and a **retry budget** (retries capped at ~10% of total calls) so retries can never dominate traffic. For tail latency specifically, **hedged requests** (fire a duplicate at p95, take the first response) often beat retries — at the cost of extra quota.

> **Keep the four mechanisms distinct** — they solve four different failures: **timeout** bounds one slow call; **bulkhead** stops one provider eating the shared pool; **breaker** stops calling a dead provider at all; **retry budget** stops your own retries amplifying a brownout. They are not interchangeable.

### 6b. Caching — the freshness/speed tradeoff

**Volatility-aware TTL:** TTL as a function of `(days_to_departure, route volume, remaining seats)`.

**Why TTL must vary:** a flight 6 months out on a high-capacity trunk route barely moves in 10 minutes → a 10-minute TTL is nearly free. A flight departing tomorrow with 3 seats left changes minute to minute → 30s or no cache. A single global TTL forces you to be either stale where it hurts most or wasteful where freshness buys nothing. Tiering spends freshness exactly where volatility is.

**Stale-while-revalidate:** on a hit past a soft-freshness threshold but within hard TTL, return it immediately *and* kick off a background refresh; the next user gets fresh data.

**Why it's the highest-leverage trick here:** it takes latency off the critical path entirely (p95 → Redis latency, ~1ms), bounds staleness by *refresh lag* rather than TTL, and — subtly but importantly — **converts user-facing traffic into background traffic**, which you can rate-limit, prioritize and schedule. Foreground traffic must be served now; background traffic you can shape.

**Negative caching:** cache "no results" and "provider errored", briefly. *Why:* otherwise routes with no flights and providers that are broken re-trigger a full fan-out on every request, which means **your most expensive code path is also your most frequently repeated one.** That's backwards, and it's exactly what saturates you during an incident.

**Cache stampede — pre-empt this follow-up.** A hot key (LHR→JFK next Friday) expires; 500 concurrent requests miss simultaneously; all 500 fan out; you breach quota in one spike and possibly take the provider down. Fixes:
- **single-flight / per-key lock** — one request refreshes, the rest wait briefly or get served stale;
- **TTL jitter** (±10%) so keys don't expire in lockstep.

**Why "indicative pricing" is a legitimate answer, not a cop-out:** the business model backs it — a search result is a **lead**, not a contract, and the price is re-validated at booking. You stay honest by *measuring*: track **price-change-at-booking rate**. If it climbs, TTLs are too long. That turns the freshness/speed tradeoff from a guess into an empirically-tuned parameter — which is the answer the interviewer is actually fishing for.

### 6c. Rate limiting toward providers

**Token bucket in Redis, executed as a Lua script for atomicity, with lazy refill.**

```lua
-- KEYS[1] = "rl:{amadeus}"    ARGV = capacity, refill_per_sec, requested
local t     = redis.call('TIME')                       -- server clock, no skew
local now   = tonumber(t[1]) * 1000 + math.floor(tonumber(t[2])/1000)
local cap   = tonumber(ARGV[1])
local rate  = tonumber(ARGV[2])
local want  = tonumber(ARGV[3])

local b      = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(b[1])
local ts     = tonumber(b[2])
if tokens == nil then tokens = cap; ts = now end

tokens = math.min(cap, tokens + math.max(0, now - ts) / 1000.0 * rate)   -- lazy refill

local allowed, wait = 0, 0
if tokens >= want then
  tokens  = tokens - want
  allowed = 1
else
  wait = math.ceil((want - tokens) / rate * 1000)
end

redis.call('HSET',    KEYS[1], 'tokens', tokens, 'ts', now)
redis.call('PEXPIRE', KEYS[1], math.ceil(cap / rate * 1000) + 1000)
return { allowed, math.floor(tokens), wait }
```

**Why Lua:** check-then-decrement is a read-modify-write. `GET` + `SET` from the client races under concurrency and lets you exceed quota exactly when traffic is highest. Lua runs atomically on the server. `INCR`+`EXPIRE` can't express refill math at all.

**Why lazy refill (store `tokens` + `ts`, compute on read):** no background refiller process, no timers, O(1) per check, and idle buckets self-clean via `PEXPIRE`.

**Why `redis.call('TIME')` instead of a client timestamp:** client clocks skew across the fleet and a fast instance would mint free tokens. The server clock is the single reference. (Requires effects-based replication — the modern Redis default.)

**Why token bucket and not a fixed window:** the bucket allows a controlled burst up to `capacity` and refills smoothly. A fixed window permits 100 calls at 0:59 and 100 more at 1:01 — 200 in two seconds while technically "in limit," exactly the pattern that gets you throttled.

**Why the limiter must be distributed at all:** you run 50 stateless instances but the provider's quota is **one global number**. Giving each instance `quota/50` fails twice — uneven traffic means some throttle while others idle (wasting quota you paid for), and autoscaling silently breaks the arithmetic.

#### Scaling the limiter

12,000 Redis ops/s just for limiting is fine for Redis but costs ~0.5ms per provider per request and creates a hot key.

- **Lease tokens in batches.** Each instance acquires ~20 tokens at once and spends them from a local counter, refilling below a watermark. **~20× less Redis traffic**; the check becomes an in-process decrement.
  *Cost:* leased-but-unspent tokens are wasted if an instance dies — mitigate with a short lease TTL and returning the remainder on graceful shutdown.
  *When not to:* if a provider's quota is tiny (5 rps), batch leasing starves other instances — go per-call there.
- **Hot-key sharding** (if leasing isn't enough): split into `rl:amadeus:{0..K-1}` with `capacity/K` each, instances pick by hash. Cost is imperfect utilization — one shard exhausted while another has tokens. Leasing is usually the better lever.

#### Providers impose several limits at once

Different dimensions need different mechanisms — a bucket cannot express "2M per day":

| Limit | Mechanism |
|---|---|
| Requests/sec | Redis token bucket (above) |
| Daily quota | `INCR` on `quota:{provider}:{YYYY-MM-DD}`, expiring at their local midnight |
| Concurrent connections | Local semaphore (the bulkhead) — it's per-instance-pool anyway |

All three must pass.

**Why "queue when the bucket is empty" is the wrong instinct:** queuing converts a rate-limit problem into a latency problem, and latency is the thing you're optimizing. Instead: serve cache if you have it, otherwise **drop that provider from this search** and mark the result partial. Degrading breadth is far cheaper than degrading latency for everyone.

**Priority without a second bucket:** foreground searches consume normally, but background cache refreshes are allowed **only when the bucket is above ~50% capacity**. One data structure, and speculative work can never starve a real user.

**Adaptive rate from provider feedback:** on `429` / `Retry-After`, apply **AIMD** — multiplicatively cut the local `rate`, then additively recover toward the configured value. *Why:* the contract number is a guess about their enforcement; their `429` is ground truth.

**When Redis is down — neither fail-open nor fail-closed.** Fail-open floods providers and gets you banned; fail-closed globally is a self-inflicted total outage. Fall back to a **local limiter set to `quota / instance_count`** (count from service discovery). Conservative, degraded, safe.

### 6d. Normalization & dedup

#### Dedup

Build a canonical identity key — `(operating carrier, flight number, departure datetime UTC, arrival, cabin, fare conditions)` — and collapse matches, keeping the best price from the most reliable source.

**Why it's harder than it sounds — codeshares:** one physical aircraft sells as BA1234, AA5678 and IB9012. Naive dedup shows three rows for one flight. Key on the **operating** carrier and flight, not the marketing one.

**Why dedup earns its complexity commercially:** the same flight from three providers should be *one* row at the best bookable price. Three near-identical rows read as clutter, make the product look broken, and push genuinely different options below the fold.

#### Price normalization

**The rule: never trust a provider's single `price` field.**

**Step 1 — decompose into components:** base fare, carrier-imposed surcharges (YQ/YR), government taxes and fees, agency/booking fee, payment-method fee, ancillaries (bag, seat).

*Why:* providers include different subsets in their headline number — that's the whole problem. Provider A quotes base fare, B quotes all-in. Naive comparison always ranks A first, the user clicks through, the price jumps 30%, and conversion craters. **Decomposition is the only way to construct a comparable total.**

**Step 2 — define exactly one canonical total, written down:**

```
total_display = base + carrier_surcharges + taxes + mandatory_fees
              (whole itinerary, all passengers, optional ancillaries excluded)
```

*Why write it down:* without one definition, "price" means subtly different things in the ranker, the UI and the booking re-validation — and nobody notices until the numbers disagree in production.

**Step 3 — the four normalization axes:**

- **Per-pax vs party total.** Some quote per adult, some the whole party. Normalize to party total, derive per-pax for display. Comparing a per-adult price against a family-of-four total is a real and embarrassing bug.
- **Currency — snapshot the FX rate per search** and store `rate_id` with it. *Why:* an FX refresh mid-search must not silently reorder results between the initial response and a streamed update — rows shuffling while the user reads is a trust-killer. It also makes "why was this ranked third?" reproducible after the fact.
- **Integer minor units, always.** All arithmetic in cents as integers, never floats. Use the **ISO-4217 exponent** — JPY has 0 decimals, KWD has 3. Hardcoding "÷100" is wrong for ~30 currencies.
- **Inclusions.** A fare without a checked bag is a *different product* from one with it. Either add the bag fee to make them comparable or segment them in the UI — don't rank them against each other as if they were the same.

**Step 4 — correct for drip pricing empirically.** Some providers reliably omit a mandatory card fee. You can't make them stop, so measure it: track the delta between quoted and actual booked total **per provider** and apply that learned correction in ranking. *Why:* you can't change provider behaviour, but you can measure its bias and price it in.

**Step 5 — validate before trusting.** Reject or flag offers where components don't sum to the total, or where price is >3σ from the route's rolling median. *Why:* cents-vs-dollars unit errors are a recurring provider-side bug, and a $4.82 fare to Tokyo at the top of your list is worse than no result at all.

```json
"price": {
  "currency": "USD",
  "total_minor": 48250,
  "components": { "base": 31000, "carrier_surcharge": 9000, "taxes": 6250, "booking_fee": 2000 },
  "original":   { "currency": "THB", "total_minor": 1720000 },
  "fx":         { "rate": 0.02805, "rate_id": "2026-08-10T14:00Z", "source": "ecb" },
  "inclusions": { "checked_bags": 1, "cabin_kg": 7, "refundable": false },
  "provider_fee_correction_minor": 900
}
```

#### Timezones

Always store UTC plus local offset. Overnight and date-line flights are the canonical bug source — arrival timestamps that appear to precede departure, or a "1-day trip" that's actually 3.

### 6e. Ranking

A weighted score over price, duration, stops, departure time, and **provider booking-success reliability**, with weights in config.

**Why booking reliability belongs in the ranking function:** you're ranking by **expected outcome, not sticker price**. A $300 fare from a provider that fails at booking 40% of the time is worth less than $320 at 99% — it wastes the user's time, burns your funnel and generates support load. Optimizing displayed price rather than *realized bookings* optimizes the wrong metric; this is the most common blind spot in the interview.

**Why weights live in config:** ranking is the primary commercial lever and will be A/B tested constantly. It must be tunable without a deploy. Extension: learned ranking with features computed offline and only scoring done online — keeps the hot path fast.

### 6f. Push vs pull for freshness

**Pull** (query on demand) is what most metasearch does: simple, reasonably current, expensive — every search costs N provider calls.

**Push** (provider streams price/availability updates) inverts the model: you maintain a **warm cache** fed by their feed, and a search becomes a *local read with no fan-out at all*.

**Why push is the real answer to fan-out amplification:** it changes the read/write ratio at the root. Instead of 12,000 outbound calls/sec scaling with *your traffic*, you have a steady update stream scaling with *inventory change rate* — a much smaller, far more predictable number, completely decoupled from your traffic spikes.

**Why hybrid is what you actually build:** not every provider offers a feed, and even with feeds you can't keep the whole long tail warm (route × date × pax combinatorics are enormous). So: **push keeps the top-N routes hot; pull handles the tail.**

**Same reasoning drives prefetch/warming:** quota is per-second and is simply *wasted* at 4am. Spending idle overnight quota to pre-warm tomorrow's popular routes converts unused capacity into peak-hour hit rate — free money against the §2 constraint.

---

## 7. Bottlenecks & tradeoffs

**Why this section exists:** it demonstrates you know the design's weak points better than the interviewer does. Volunteering them reads as maturity; being *caught out* on them reads as inexperience.

1. **The slowest provider dominates perceived latency.** With 20 providers, "at least one is slow" is the *majority* case (~64%), not the tail. → deadlines + progressive results. Accepted cost: some searches never show every provider's inventory.

2. **Staleness vs booking failure.** Too-long TTL → users click a fare that's gone → booking failure and a support ticket. Too-short → quota breach and 8-second searches. There is no correct static value; tune per route segment, monitor via price-change-at-booking.

3. **Provider quotas are a hard external wall.** The tradeoff that most distinguishes this problem: **you cannot scale your way out of it.** Adding servers makes it *worse*. The only levers are caching, limiting, breaking and push feeds. Say this explicitly — it proves you understand why the design looks the way it does.

4. **Fan-out amplification, and why hit rate is *the* lever.** Note the non-linearity: 95% → 97.5% hit rate doesn't improve things by 2.5% — it **halves** outbound volume (5% → 2.5% of requests escaping). Near the top of the range, small hit-rate gains produce huge load reductions, which is why cache tuning outranks essentially every other optimization here.

5. **Redis is now your most dangerous component** — and not in the usual way. **A full cache flush is an instant total outage**, because 100% of traffic converts to fan-out and you breach every provider quota simultaneously. So treat the cache as a protected dependency: cluster with replicas, warm restarts, **gradual re-warm** after a cold start, and aggressive **load shedding while the cache is cold** (serve fewer providers, or reject some traffic, rather than melting your partners). *"How do you recover from a cache flush?"* is a very likely follow-up.

6. **Cost, not just capacity.** GDS calls often cost real money per search, and contracts frequently impose **look-to-book ratio** penalties — search too much relative to bookings and you pay more or get cut off. So caching is simultaneously a latency lever, a quota lever and a **margin** lever. This shows commercial awareness, which is unusual and lands well.

**What you'd instrument** (the "how do you know it works" answer):
- per-provider p50/p95, timeout rate, error rate
- **cache hit rate per provider** — aggregate hit rate hides a single provider quietly falling off a cliff
- **result completeness** — % of searches that got all providers. This is your real quality metric and it's *invisible* if you only watch latency
- look-to-book ratio, and price-change-at-booking

**Next steps / resilience:** provider health dashboard, per-provider SLO tracking, adaptive timeouts, A/B-tunable ranking, warm-cache prefetch for top routes ahead of peak.

---

## 8. Language / runtime choice

**Go for the orchestrator and adapter layer; Python offline; Java/Kotlin the one genuinely credible alternative.** The reasoning is derived from this workload, not from general preference — which is the part interviewers actually care about.

### Why Go

**1. `context.Context` *is* the deadline model from §6a.** The core mechanic — a global deadline that abandons in-flight calls, with per-provider timeouts nested inside it — maps directly onto Go's context tree. Cancellation propagates: the deadline fires, all 20 in-flight HTTP calls abort, sockets return to the pool immediately.

```go
func (o *Orchestrator) Search(ctx context.Context, q Query) Results {
    ctx, cancel := context.WithTimeout(ctx, 1500*time.Millisecond)  // global deadline
    defer cancel()

    var wg sync.WaitGroup
    out := make(chan ProviderResult, len(o.adapters))

    for _, a := range o.adapters {
        wg.Add(1)
        go func(a Adapter) {                      // ~4KB stack, not an OS thread
            defer wg.Done()
            pctx, pcancel := context.WithTimeout(ctx, a.Timeout())  // per-provider timeout
            defer pcancel()
            if r, err := a.Search(pctx, q); err == nil {
                out <- r
            }
        }(a)
    }
    go func() { wg.Wait(); close(out) }()

    return collectUntil(ctx, out)   // returns on deadline OR completion, whichever first
}
```

That's the whole §6a fan-out with correct cancellation. In Node it's `AbortController` threaded manually through every adapter; in Python asyncio, cancellation semantics around `gather` are a well-known footgun.

**2. The concurrency number, by Little's Law.** Peak with a cold cache: 12,000 outbound calls/s × ~1.5s average duration = **~18,000 concurrent in-flight outbound connections**, ~900 per instance across a 20-node fleet. Goroutines make that a non-event. Pre-Loom thread-per-request Java would die on it.

**3. The hidden CPU cost is parsing.** GDS responses are frequently multi-megabyte XML, normalized at thousands per second. That's real CPU, and in Go it's *parallel across cores within one process*. **In Node it blocks the event loop** — a 3MB XML parse stalls every other in-flight search on that process. The fix is `worker_threads` or process-per-core, which leads directly to:

**4. Node's process-per-core model actively fights §4c, §6a and §6c.** All three depend on *in-process* state: the config hashmap, the local circuit-breaker window, and leased rate-limit tokens. Eight cores means eight copies per machine, each seeing 1/8 the traffic — which worsens the exact problem flagged in §6a (the local breaker never accumulates enough samples to trip) and multiplies leased-token waste 8×. Go keeps **one shared in-process state per machine**, which is what those designs assume.

**5. Predictable GC.** This is a tail-latency system throughout. Go's concurrent low-pause collector keeps p95 stable; you cannot afford a stop-the-world pause landing inside a 1.5s deadline budget.

**6. Static typing across 20 heterogeneous schemas.** Normalization (§6d) is exactly where a type system pays — `Price{TotalMinor int64, Currency string}` enforces integer minor units at compile time, preventing the float and per-pax bugs that section warns about.

### The honest alternative: Java / Kotlin

Genuinely competitive, and what many real travel companies actually run:
- **Virtual threads (Loom)** close the concurrency gap with Go entirely.
- **Resilience4j** provides breakers, bulkheads, rate limiters and retry budgets off the shelf, battle-tested — in Go you assemble or hand-roll these.
- **The decisive practical point:** airline/GDS integration SDKs are often Java-first, and SOAP / WS-Security tooling (Sabre, older Amadeus surfaces) is far more mature on the JVM. If half your providers are SOAP, this can outweigh everything above.

Cost: heavier memory footprint and GC tuning you actually have to do.

### Where the others belong

| | Verdict |
|---|---|
| **Python** | **Offline, never the hot path.** Ranking model training, provider reliability scoring, TTL tuning from price-volatility analysis, look-to-book analytics. GIL + slow parsing rule it out for fan-out at 600 rps. |
| **Node / TypeScript** | Fine for a thin **BFF/edge tier** serving SSE to browsers, and the fastest language to write adapters in. Reasonable for the whole system at ~1/10th this scale or if every provider returns modest JSON. Degrades badly precisely where this design is hardest: large XML payloads and per-machine shared state. |
| **Rust** | Overkill. You're writing 20+ adapters against messy third-party schemas; iteration speed on that surface matters more than the last 20% of performance. |

### Realistic polyglot split

```
Go            → orchestrator, adapters, rate limiter, breaker      (hot path)
Python        → ranking model training, TTL/volatility tuning,
                provider reliability scoring, analytics            (offline)
TypeScript    → web client; optional BFF for SSE fan-out           (edge)
```

**Interview caveat:** if asked this live, answer in two sentences — *"Go, because the fan-out-with-deadline pattern maps onto goroutines plus context cancellation, and normalization is CPU-heavy enough that a single-threaded runtime would block"* — then get back to the design. Interviewers rarely care which language; they care that the reason is derived from the workload rather than from familiarity.

---

## The through-line

**You don't control your data, so you control your exposure to it.** Caching reduces how often you touch providers; timeouts and deadlines bound how long a bad provider can hurt you; circuit breakers stop touching broken ones entirely; rate limiters keep you inside contractual limits; progressive results make the whole thing feel fast anyway. Sections 1–7 are seven views of that one idea.
