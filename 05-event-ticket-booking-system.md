# 5. Event Ticket Booking System (Ticketmaster / BookMyShow style)

**One-liner:** A system where **we own the inventory**, so correctness is enforced *in our own datastore*: users reserve seats via a **short-TTL hold** backed by **atomic, guarded decrements** (or unique constraints) to prevent oversell, a **virtual waiting room / admission queue** absorbs flash-sale spikes, rate limiting + bot protection defend against abuse, and reads scale via caching while writes stay strongly consistent on the seat map.

The interviewer's explicit sub-questions map cleanly onto sections below — answer them in order and you've covered the whole prompt.

---

## 1. Requirements & scale

**Functional**
- Browse events, view seat map / availability.
- Reserve (hold) → pay → confirm ticket(s).
- Support both **assigned-seat** (specific seats) and **general-admission** (count-based) events.

**Non-functional**
- **No double-booking, no overselling** — the correctness bar.
- **Flash-sale spikes:** a hot concert = huge burst of concurrent buyers on a tiny inventory (the defining challenge). Read-heavy browsing, write-hot at on-sale moment.
- Low latency browsing; strong consistency on the seat state.

**Scale intuition:** browsing traffic is large but cacheable; the *dangerous* load is thousands of users hitting **the same few hundred seats in the same few seconds**. Design for that hotspot.

## 2. API design

```
GET  /events/{id}                          # event + cached availability summary
GET  /events/{id}/seats                    # seat map (cached, near-real-time)
POST /events/{id}/holds  { seatIds | qty } # reserve with TTL -> { holdId, expiresAt }   (idempotent)
POST /holds/{holdId}/confirm { paymentToken, idempotencyKey } -> ticket(s)
DELETE /holds/{holdId}                      # release early
```

## 3. Data model / storage choice

```
events(event_id, venue_id, on_sale_at, ...)
seats(event_id, seat_id, section, status ENUM('available','held','booked'),
      held_by, hold_expires_at, version)          -- assigned-seat model
inventory(event_id, section, total, available)     -- general-admission model
bookings(booking_id, user_id, event_id, seat_ids[], status, idempotency_key UNIQUE, created_at)
UNIQUE(event_id, seat_id) on the confirmed-tickets table   -- hard oversell backstop
```

**Storage choice — say why:**
- **Seat inventory → relational (SQL).** You need **ACID transactions and strong consistency** to guarantee no oversell. This is the textbook case *against* eventually-consistent NoSQL for the inventory of record. Availability *reads* can be cached/denormalized, but the **write of truth is SQL**.
- **Redis** for holds/TTL and hot availability cache and rate limiting.
- **Read replicas / cache** for the seat-map browse traffic (read-heavy, tolerates slight staleness).

## 4. High-level architecture

```
                 ┌── CDN / cache (event pages, seat map reads) ──┐
Client ──► LB ──► (optional) Virtual Waiting Room ──► API (stateless)
                                                        │
                        ┌───────────────────────────────┼──────────────────┐
                        ▼                                ▼                   ▼
                  Hold Service (Redis TTL)        Inventory Service     Bookings DB (SQL,
                  atomic reserve/release          (atomic decrement)    replicated) + outbox
                                                                            │
                                            Payment Service (idempotent) → Kafka →
                                            workers: hold-expiry sweeper, ticket issue,
                                            notifications, analytics
```

---

## 5. Sub-question: How is concurrency handled?

The whole design *is* the concurrency answer. Three coordinated mechanisms:
1. **Hold with TTL** so the seat is taken off the market the instant a user selects it, without holding a DB lock across their payment think-time.
2. **Atomic state transition** to move a seat `available → held` — only one concurrent request can win (see next section for the exact mechanism).
3. **Idempotency** so client retries / double-clicks during the frenzy don't create duplicate holds or bookings.

Say explicitly: **the read-modify-write on seat status is the critical section**, and it must be atomic at the datastore, not in application memory (app memory doesn't help across N stateless instances).

## 6. Sub-question: How do we avoid two users booking the same seat?

Make `available → held` an **atomic, guarded write**. Options (know all four, pick per contention):

- **Conditional UPDATE (recommended default):**
  ```sql
  UPDATE seats SET status='held', held_by=?, hold_expires_at=now()+interval '8 min'
  WHERE event_id=? AND seat_id=? AND status='available';
  ```
  Single-statement atomicity + the `status='available'` guard ⇒ exactly one winner; check `rowsAffected==1`. Loser gets "seat taken."
- **Optimistic locking:** `version` column, `WHERE version=?`, retry on conflict. Great for low contention.
- **Pessimistic locking:** `SELECT ... FOR UPDATE` on the seat row. Correct but serializes hot rows — use sparingly.
- **Distributed lock in Redis** (`SET NX` with TTL, e.g. Redlock-style) for the hold, with SQL as durable truth on confirm.
- **Backstop:** `UNIQUE(event_id, seat_id)` on the confirmed-tickets table so even if two racing paths slipped through, the DB rejects the second insert. Defense in depth.

For **general admission** (count, not seats): atomic decrement
```sql
UPDATE inventory SET available = available - :qty
WHERE event_id=? AND section=? AND available >= :qty;   -- guard prevents going negative
```
or a Redis `DECRBY` guarded by a Lua script.

## 7. Sub-question: How do we prevent overselling?

Overselling = the count goes below zero / more tickets issued than exist. Prevent it by ensuring **the decrement is atomic and guarded** (above) and by **defense in depth**:
- The `WHERE available >= qty` / `status='available'` guard means the DB itself refuses to oversell — never `read count → check in app → write` (that's the classic race).
- **Single source of truth** for inventory (one SQL row per seat/section); caches are *derived*, never authoritative for the decrement.
- **Reserved buffers** for known integration lag (e.g. hold back a few % if syncing with an external box office).
- **Reconcile expired holds** promptly (sweeper) so held-but-unpaid seats return to inventory and aren't lost.
- **Idempotency** so a retried confirm doesn't issue a second ticket for the same hold.
- If you ever use Redis for the fast-path count, **reconcile Redis ↔ SQL** and treat SQL as the ledger.

## 8. Sub-question: How do we protect from abuse and attacks?

- **Virtual waiting room / admission queue:** at on-sale, admit users into the buying flow at a **controlled rate** (token/queue in Redis). This is *the* flash-sale defense — it converts an un-survivable spike into a steady stream, protects the DB, and gives fair ordering. Users see "you're in line, position N."
- **Rate limiting** per user / IP / API key (token bucket, distributed via Redis) → blocks rapid-fire scripted requests; `429` + `Retry-After`.
- **Bot / scalper defense:** CAPTCHA at high-value steps, device fingerprinting, account age/verification requirements, **purchase quantity caps** per user/payment method/event, velocity checks.
- **Idempotency + hold TTL** also blunt abuse (can't spam-hold the whole map: holds expire; per-user hold limits).
- **DDoS / edge protection:** WAF, CDN, edge rate limiting before traffic reaches the app.
- **Auth on state-changing endpoints**; signed, expiring hold tokens so holds can't be forged/extended.
- **Fraud checks** at payment (unusual patterns, mismatched geo).

## 9. Sub-question: What improvements make it more resilient & scalable?

- **Separate read and write paths (CQRS-ish):** browse/seat-map served from cache + read replicas (huge, cacheable, staleness-tolerant); the hot write path (hold/confirm) kept lean and strongly consistent. Reads never contend with the critical section.
- **Queue-based async** for everything off the critical path: ticket PDF generation, emails, analytics, syncing → smooths spikes, isolates failures, DLQ for retries.
- **Shard/partition by event:** a mega-event's load is isolated to its shard; noisy neighbors don't sink the platform. Consistent hashing to add capacity without mass reshuffle.
- **Horizontal, stateless app tier** + auto-scaling behind the LB; **DB read replicas** + failover; multi-AZ.
- **Graceful degradation:** if the seat-map service is degraded, still allow best-available booking; if payments are slow, hold the seat and confirm async.
- **Circuit breakers / timeouts / bulkheads** around payment + any external dependency.
- **Observability:** RED metrics, hold→confirm funnel, oversell alarms (should be *zero* — alert on any negative inventory), waiting-room throughput.
- **Pre-warm & load-test** before big on-sales; capacity plan for the known peak.

---

## 10. Tradeoffs to name out loud
- **Strong consistency on inventory vs latency/throughput:** you deliberately pay latency (atomic guarded writes, possibly serialized hot rows) to guarantee **zero oversell** — the one thing you can't get wrong.
- **Hold TTL length:** too short frustrates real buyers mid-payment; too long starves inventory and helps scalpers. Tune ~5–10 min.
- **Redis speed vs SQL durability:** Redis for the fast hold/count + waiting room, SQL as the durable ledger, with reconciliation bridging them.
- **Waiting room fairness vs UX:** a queue is less "instant" but is the only thing that keeps the system up and fair under a true flash sale.

**Contrast with Q4 (flight booking):** here **we own inventory**, so correctness is enforced in *our* DB (atomic decrements, locks, unique constraints). In flight booking the **airline owns inventory**, so correctness shifts to **holds + saga + reconciliation with an external system of record**. Same primitives (TTL hold, idempotency, saga), different locus of truth. Saying this comparison out loud is a strong senior signal.
