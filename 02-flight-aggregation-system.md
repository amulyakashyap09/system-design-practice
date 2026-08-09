# 2. Flight Aggregation System (flight search across many providers)

**One-liner:** A search service that fans out a query to many airline/GDS/aggregator providers in parallel with per-provider timeouts and circuit breakers, normalizes and merges their responses, caches results with short freshness-aware TTLs, and returns ranked results — degrading gracefully to partial results when some providers are slow or down.

This is the classic Agoda-shaped problem: **you don't own the inventory; you integrate with many external providers whose data is slow, rate-limited, and constantly changing.** The design is dominated by that reality.

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

**Questions to ask:** how many providers, and what are their SLAs/rate limits? how fresh must prices be (can we cache 5 min)? is this search-only or does it also feed booking (freshness bar is higher if it feeds booking)? push (providers stream updates) or pull (we query on demand)?

## 2. Back-of-envelope
- Say 10M searches/day ≈ **~120 searches/sec** avg, ~600/sec peak.
- Each search fans out to, say, 20 providers → **2,400–12,000 outbound provider calls/sec** if uncached. **This is why caching + rate limiting toward providers is mandatory** — you'll blow past provider quotas instantly otherwise. (This directly sets up question 3.)
- Cache hit ratio target: high on popular routes (a few routes = most traffic → long-tail caching).

## 3. API design

```
POST /search
  body: { origin, destination, departDate, returnDate?, pax:{adt,chd,inf}, cabin }
  -> { searchId, results:[...], partial:bool, completedProviders, pendingProviders }

GET  /search/{searchId}          # poll for late-arriving provider results (progressive)
GET  /search/{searchId}/stream   # SSE/WebSocket: push results as each provider returns
```

Design choice: **progressive results.** Return whatever's ready fast (cache + fast providers), then stream in slow providers. Don't block the user on the slowest provider. This is how real flight metasearch feels responsive despite 8s GDS calls.

## 4. Data model / storage

- **Provider adapters config:** per-provider endpoint, auth, rate limit, timeout, method mapping, response schema mapping.
- **Cache (Redis):** key = normalized `(origin, dest, date, pax, cabin, provider)` → normalized results + `fetched_at`. Short TTL (minutes) tuned per route volatility.
- **Normalized result schema:** unified `Flight { segments[], price{amount,currency}, fareClass, baggage, provider, deeplink/offerToken, fetchedAt }`.
- No heavy primary DB on the hot path — this system is a **read/aggregation cache in front of external APIs**, plus config and logs. Analytics/pricing history can go to a columnar store offline.

## 5. High-level architecture

```
Client
  │  POST /search
  ▼
API / Search Orchestrator (stateless)
  ├─ 1. normalize query, build cache key
  ├─ 2. cache lookup (Redis) — return hits immediately
  ├─ 3. fan-out to Provider Adapter layer for misses (parallel, bounded)
  │        ├─ Adapter A ─(rate limiter, timeout, circuit breaker, retry)→ Provider A
  │        ├─ Adapter B ─ ... → Provider B
  │        └─ Adapter N ─ ... → Provider N
  ├─ 4. normalize + merge + dedup + rank
  ├─ 5. write results back to cache with TTL
  └─ 6. return partial now, stream the rest
```

Each **provider adapter** encapsulates that provider's quirks: auth, correct HTTP method/semantics, request/response schema mapping, its rate limit, its timeout. Adding a provider = adding an adapter, not touching core logic (open/closed).

## 6. Deep dives

### Fan-out with graceful degradation (the core skill here)
- Call all providers **in parallel** with a **hard per-provider timeout** (e.g. 3s). Use `Promise.allSettled`-style semantics: collect whoever finishes; the slow/failed ones are simply omitted.
- **Overall deadline:** e.g. return best-effort at 2s regardless; late providers fill in via the poll/stream endpoint.
- **Circuit breaker** per provider: if a provider is erroring/timing out, stop calling it for a cooldown and serve cache — protects your latency and the provider both.

### Caching strategy — the freshness/speed tradeoff
- Cache normalized per-provider results with **short, volatility-aware TTL.** Popular, stable routes → longer TTL; volatile/last-minute fares → very short or no cache.
- **Stale-while-revalidate:** serve cached results instantly, trigger a background refresh so the *next* user gets fresh data. Keeps p95 low.
- Be explicit that cached price is *indicative*; final price is re-validated at booking (a search hit is a lead, not a contract). This is how the industry reconciles "fast search" with "accurate booking."

### Rate limiting toward providers (bridges to Q3)
- Each adapter enforces the provider's quota with a **token bucket** so your fan-out never exceeds what the provider allows. When the bucket is empty, serve cache or drop that provider from this search rather than getting throttled/banned.
- Aggregate across your whole fleet via a **distributed** limiter (Redis token bucket), because horizontal scaling means many app instances share one provider quota.

### Normalization & dedup
- Providers return the same physical flight in different shapes/currencies. Normalize to a canonical schema; **dedup** the same flight offered by multiple providers, keeping the best price / most reliable source. Handle currency conversion and timezone/date-line correctness (a real source of bugs in flight systems).

### Ranking
- Score by price, duration, stops, departure time, and **provider booking-success reliability** (a cheap fare from a provider that fails at booking is worth less). Rank offline-tunable.

### Push vs pull for freshness
- Pure pull (query on demand) is simplest and what most metasearch does. If a provider supports **push** (streams price/availability updates), maintain a warm cache updated by their feed → near-real-time without per-search fan-out. Hybrid is common.

## 7. Bottlenecks & tradeoffs
- **Slowest provider dominates perceived latency** → solved by deadlines + progressive results.
- **Cache staleness vs booking failures:** too-long TTL → users click a fare that's gone → booking failure. Tune per route; always re-validate at booking.
- **Provider rate limits are a hard external constraint** → caching + limiting + circuit breaking, not "just scale up."
- **Fan-out amplification:** 1 search = 20 calls. Without caching your outbound volume is unmanageable → cache hit ratio is the primary lever.

**Next steps / resilience:** provider health dashboard, per-provider SLO tracking, adaptive timeouts, A/B tunable ranking, and a warm-cache prefetch for top routes ahead of peak.
