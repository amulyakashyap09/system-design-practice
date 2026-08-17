One-liner: ingest everything into a partitioned Kafka spine, fan out to channel-specific dispatchers behind provider abstractions, and keep rate-limiting, scheduling, dedup, and retries as separate concerns off to the side.

Here's the whiteboard:**Capacity (back-of-envelope)**
![Architecture Diagram](./diagrams/08-push-notification-system/08-push-notification-diagram.png)
- Volume: 50M × 5 = 250M notifications/day → ~2.9K/s average, ~16.7K/s at peak (1M/min). Peak is ~6× average, so provision for burst, not steady state.
- Storage: notification metadata + status logs at ~1KB each → ~250GB/day of write-heavy, short-TTL data. Keep hot status in a fast store (Cassandra/DynamoDB), age the rest to cold storage.
- The queue is the shock absorber. Producers never block on delivery; the flash-sale spike lands in Kafka and drains at whatever rate downstream can sustain.

**Request path, component by component**
- Ingestion (gateway + notification service): authenticates the caller, validates payload, resolves user + device tokens, stamps a notification type (transactional / promotional / system) and an idempotency key. Type drives priority — transactional and system alerts go to a high-priority topic, promotional to a bulk topic so a promo flood never starves an OTP.
- Kafka: partition by user_id (ordering per user) or by channel (isolation). Separate topics per priority class. Retention gives you replay for free.
- Scheduler: future/recurring requests persist to a DB; a time-wheel or a partitioned "due" scan enqueues them into Kafka when the fire time arrives. Keeps scheduling logic out of the hot path.
- Dispatch workers: stateless consumers, autoscaled on consumer lag. They enrich against the lookup services, render the template, then hand off to the right channel dispatcher.
- Lookup services (the policy layer): user preferences/opt-outs, rate limiter, template store, dedup. All are cache-first (Redis) so per-message lookups stay sub-millisecond.
- Channel dispatchers + providers: thin adapters that normalize to each provider's API and own provider-specific retry/circuit-breaking. Provider abstraction lets you multi-home (e.g. failover SendGrid→SES) without touching workers.

**How each requirement maps**
- Multi-channel: dispatcher-per-channel behind a common interface; adding WhatsApp is a new adapter, not a pipeline change.
- Notification types: priority topics + separate consumer pools so bulk promos can't delay transactional.
- Scheduled delivery: dedicated scheduler service feeding Kafka; no polling in workers.
- Rate limiting: token-bucket per (user, category) in Redis, checked in the lookup layer — promotional capped daily, transactional exempt.
- Retry: dispatcher classifies failures (retryable vs permanent), re-enqueues retryables to a delay/retry topic with exponential backoff + jitter; exhausted messages land in the DLQ for inspection/replay.

**Non-functional levers**
- Scalability: everything between gateway and providers is stateless and horizontally scaled; Kafka partitions are the scaling unit. Peak is absorbed by lag, not by dropping.
- Availability: no single choke point — multi-AZ Kafka, replicated stores, provider failover. A dead provider degrades one channel, not the system.
- Reliability: Kafka + consumer offset commits after successful hand-off = at-least-once. For exactly-once *effect*, dedup on idempotency_key at ingestion and again pre-send (a Redis SETNX / Bloom filter over a TTL window) — you get at-least-once transport with idempotent delivery, which is the realistic target. True exactly-once transport is expensive; reserve it for cases that genuinely need it.
- Latency: transactional path skips the bulk queue and rate limiter; keep lookups in-cache; treat delivery as async from the caller's perspective.

**Two things worth deciding explicitly in an interview**
- Partition key: user_id (ordering, but hot-user skew) vs channel (even load, no per-user ordering). State the tradeoff.
- Dedup window: exactly-once "feel" is really "at-least-once + idempotency over a bounded TTL." Be honest that it's not true EOS, and size the window to your retry horizon.


# Detailed Explanation

Good — I'll go stage by stage down the pipeline. Each section has a focused diagram plus the mechanism, the key decisions, and the line you'd actually say in an interview.

## 1. Ingestion — turning a request into a routed, deduplicated message

![Architecture Diagram](./diagrams/08-push-notification-system/08-push-notification-diagram-1.png)

One-liner: the ingestion layer's job is to make the message *safe to process* — authenticated, valid, classified, and stamped — before it ever touches the queue.- Gateway vs service split: the gateway does cheap, stateless rejection (bad tokens, coarse per-caller quotas) so garbage never reaches your business logic. The notification service does the expensive, stateful work. Keep them separate so a misbehaving client is stopped at the door.
- Resolve is where opt-outs and device-token lookups happen. A user with no valid device token for `push` shouldn't generate a push message — resolve to the channels that are actually deliverable, or fall back (push → email).
- Classification is load-bearing: `type` decides the topic, the priority, and whether rate limiting even applies. Get it wrong and a promo can ride the transactional lane.
- The idempotency key is stamped *here, once*, by hashing a caller-supplied `dedupe-id` (or business key like `order_id + event`) with `user` and `type`. This is the anchor for exactly-once *effect* later — if you don't create it at ingestion, you can't dedup downstream.
- Interview line: "Ingestion is synchronous and fast; it validates, classifies, stamps an idempotency key, and hands off to Kafka. Everything after is async."

## 2. Kafka topology — priority isolation and partitioning

![Architecture Diagram](./diagrams/08-push-notification-system/08-push-notification-diagram-2.png)

One-liner: separate topics per priority class give you isolation; partitioning gives you scale and ordering — and the two choices trade off against each other.- Why separate topics, not just separate partitions: consumer groups bind to topics. Two topics = two independent consumer pools you can scale, deploy, and fail independently. During a flash sale the bulk pool's lag balloons; the high-priority pool doesn't even notice because it's a different topic with its own consumers. One topic with a "priority field" can't give you that isolation — a slow promo batch sits in the same partitions as your OTPs.
- Partition count = your max consumer parallelism. One partition is consumed by exactly one consumer in a group, so if you have 3 partitions you can't usefully run 10 consumers. Size partitions for *peak* throughput (16.7K/s ÷ per-consumer rate), then add headroom — repartitioning later is painful.
- The partition-key decision is the classic tradeoff:
  - `user_id`: all of a user's notifications land on one partition → strict per-user ordering (welcome before order-confirm before shipping). Risk: a celebrity/hot user creates a hot partition.
  - `channel` or random: even load, no hot partitions, but you lose per-user ordering.
  - Practical answer: key by `user_id` for correctness, and mitigate skew with a hashed composite key or by splitting known hot users. Say the tradeoff out loud — interviewers are listening for exactly this.
- Retention buys you replay: if a downstream bug corrupts sends, you reset offsets and reprocess. Set retention to at least your worst-case incident-recovery window.

## 3. Dispatch workers + the policy layer

![Architecture Diagram](./diagrams/08-push-notification-system/08-push-notification-diagram-3.png)

One-liner: workers are stateless consumers that run each message through an ordered gauntlet of cache-first checks, and any check can short-circuit the send.- Order matters and it's deliberate: cheap, high-rejection checks go first so you don't waste work. Opt-out and rate-limit reject a lot and cost almost nothing — do them before the more expensive template render. Dedup goes last, right before the send, so it catches duplicates introduced by *redelivery* (not just at ingestion).
- Stateless is the whole point: workers hold no session state, so you scale them purely on Kafka consumer lag and kill/restart them freely. All state lives in Redis and the datastores.
- Cache-first or you die at peak: at 16.7K/s every message does 3–4 lookups → ~50–65K lookups/s. That has to be Redis-speed. Preferences and templates are read-heavy and rarely change → cache with generous TTLs and invalidate on write.
- Template render belongs here, not at ingestion: it's per-recipient (name, locale, currency) and you want the freshest data at send time, not stale data captured minutes earlier when it was enqueued.
- Interview line: "The worker is a stateless consumer running an ordered policy gauntlet — prefs, rate limit, render, dedup — all Redis-backed, each able to short-circuit. Cheapest and most-rejecting checks first."

## 4. Rate limiter — the token bucket data model

![Architecture Diagram](./diagrams/08-push-notification-system/08-push-notification-diagram-4.png)

One-liner: a per-(user, category) token bucket in Redis caps promotional volume while letting transactional traffic through untouched.- Why token bucket over a fixed counter: a bucket allows controlled bursts (a user can get 3 promos in a burst but only N/day total) and refills continuously, avoiding the "midnight thundering herd" you get when a daily counter resets for everyone at once.
- The key design is the interview meat: `rl:{user_id}:{category}` with two fields — `tokens` and `last_refill_ts`. On each check you lazily refill: `tokens = min(capacity, tokens + elapsed * refill_rate)`, then try to decrement. Lazy refill means no background job sweeping millions of buckets — you only compute refill when a message actually arrives.
- Make it atomic: the read-refill-decrement must be a single Redis Lua script (or `INCR`-based algorithm), or two workers racing will both spend the last token. This is the classic concurrency bug they'll probe.
- Category scoping is what enforces "promos capped, OTPs never blocked": transactional and system skip the bucket entirely. You can also layer a global per-tenant limit above the per-user one to protect providers.
- Set a TTL on the key equal to the window so idle users' buckets self-evict — keeps Redis memory bounded across 50M users.
- Interview line: "Per-user, per-category token bucket in Redis with lazy refill, mutated atomically via Lua. Transactional bypasses it. TTL-evict idle keys to bound memory."

## 5. Delivery semantics — at-least-once transport, exactly-once effect

![Architecture Diagram](./diagrams/08-push-notification-system/08-push-notification-diagram-5.png)

One-liner: you can't cheaply guarantee exactly-once *delivery*, so you guarantee at-least-once transport and make the send *idempotent* — the dedup check turns "maybe twice" into "exactly once as the user sees it."- The ordering is the whole trick: send *before* committing the offset. If the worker crashes after sending but before committing, Kafka redelivers the message — the dedup check sees `K` already present and skips. You never lose a message (at-least-once) and the user never sees a duplicate (idempotent effect).
- Why not exactly-once transport: true EOS across Kafka *and* an external provider (FCM, Twilio) is impossible — the provider isn't in your transaction. The provider might deliver but time out on the ack, and you genuinely can't tell "sent" from "not sent." So you accept at-least-once and dedup.
- The dedup store is a bounded-TTL set, not a permanent log: `SETNX K` with a TTL covering your maximum retry horizon (e.g. 24–48h). After that the key expires — you only need to catch duplicates within the redelivery window, not remember every notification forever. For 50M×5/day this keeps the working set manageable; a Bloom filter or Redis with eviction handles the volume.
- Edge case worth naming: the send succeeds but persisting `K` fails. Then a redelivery re-sends. Mitigate by making the `SETNX` *before* the send claim the key (reserve), and finalize after — the reserve-then-confirm pattern narrows the window. Be honest that it's "at-least-once with a small duplicate probability," not a mathematical guarantee.
- Reserve exactly-once effort for messages that need it (payment receipts, OTPs). Marketing pushes can tolerate a rare dupe and don't justify the extra store writes.
- Interview line: "At-least-once from Kafka by committing offsets only after a successful send, made idempotent by a TTL-bounded dedup key checked before the provider call. Exactly-once *transport* to a third-party provider is impossible, so I target exactly-once *effect*."

## 6. Retry and DLQ topology

![Architecture Diagram](./diagrams/08-push-notification-system/08-push-notification-diagram-6.png)

One-liner: failures are classified, retryable ones flow through tiered delay topics with exponential backoff, and anything that exhausts its attempts lands in a dead-letter queue for humans.- Classify before you retry, always: retrying a permanent failure (invalid device token, unsubscribed number, malformed payload) is pure waste and can look like abuse to the provider. Bad tokens should be dropped *and* fed back to prune the user's token list. Only transient failures (5xx, timeout, rate-limited-by-provider) are retryable.
- Don't retry in-process — use delay topics. Blocking a worker thread with `sleep(30m)` holds the Kafka partition and stalls everything behind it. Instead publish to a delay topic keyed by tier (retry-1m, retry-5m, retry-30m); a delayed consumer or the topic's visibility timeout releases the message when it's due, and it re-enters the normal flow. The worker stays free.
- Exponential backoff **with jitter** is non-negotiable: if a provider blips and 100K messages all fail at once, retrying them all at exactly +1m recreates the same thundering herd. Jitter spreads them. Pair this with a circuit breaker per provider — if failures spike, stop hammering and shed to the retry tier or a backup provider.
- The DLQ is an operational tool, not a graveyard: it should alert (a full DLQ means something is systemically broken), preserve the full message + failure reason for debugging, and support replay once the root cause is fixed. Retention long enough to actually investigate.
- Idempotency ties it together: because retries re-run the whole worker path, the dedup check from section 5 is what stops a retry from double-sending a message that actually *did* go out but whose ack was lost.
- Interview line: "Classify first — only transient failures retry. Retries go through tiered delay topics with exponential backoff plus jitter and a per-provider circuit breaker, never an in-process sleep. Exhausted messages hit a DLQ that alerts and supports replay."

---

That's the full pipeline end to end. The five drill-downs an interviewer is most likely to push on — partition key skew, atomic token-bucket updates, exactly-once effect vs transport, in-process-sleep vs delay-topic retries, and the send-then-commit ordering — are each called out in the "interview line" for that section, so those are the ones worth having crisp.