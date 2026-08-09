# 4. Flight Booking System

**One-liner:** A booking service that turns a search offer into a confirmed ticket via a **saga** — reserve seat with a short TTL hold, re-validate price/availability with the airline, charge payment idempotently, confirm with the airline, then persist the booking — with **compensating actions** (release + refund) on any failure and an async **reconciliation** loop to keep our state in sync with the airline of record.

The four deep dives they called out — **booking contention, payments, failure handling, aggregator sync** — are the meat. Structure your answer around them.

---

## 1. Clarify requirements

**Functional**
- Convert a selected flight offer into a booking (PNR).
- Handle passenger details, seat/fare selection, payment.
- Confirm with the airline/aggregator (they are the **source of truth** for inventory — we don't own the seat).
- Booking lifecycle: `INITIATED → HELD → PAID → CONFIRMED / FAILED / CANCELLED`.
- Cancellation/refund.

**Non-functional**
- **Correctness > raw latency** here (money + inventory). No double-charge, no double-book, no lost bookings.
- Strong consistency on a booking's state; durability of every booking record.
- Resilient to airline/payment provider slowness and failure.

**Key clarifying point:** unlike an event-ticket system where *we own inventory*, here the **airline owns the seat**. We can only *request* a booking and must **reconcile** with their answer. This reframes contention: it's less "lock our row" and more "safely coordinate a distributed transaction with an external system of record + handle ambiguous outcomes."

## 2. API design

```
POST /bookings            { offerId, pax[], contact }         -> { bookingId, status:HELD, holdExpiresAt }  (idempotent via Idempotency-Key)
POST /bookings/{id}/pay   { paymentToken, idempotencyKey }    -> { status: PAID|CONFIRMED|FAILED }
GET  /bookings/{id}                                            -> current state
POST /bookings/{id}/cancel                                     -> refund saga
```
Every mutating call carries an **idempotency key**; retries return the prior result, never re-execute.

## 3. Data model

```
bookings(booking_id PK, user_id, offer_id, airline_ref/PNR, status, price, currency,
         hold_expires_at, created_at, version)
booking_pax(booking_id, name, dob, ...)
payments(payment_id PK, booking_id, provider_ref, amount, status, idempotency_key UNIQUE)
booking_events(booking_id, event, payload, at)     # audit / saga log / outbox
```
- **SQL** — you need ACID transactions and a durable, auditable state machine per booking.
- `idempotency_key UNIQUE` on payments is a hard DB-level guard against double-charge.
- `booking_events` doubles as the **transactional outbox** (state change + event emitted atomically).

## 4. High-level architecture

```
Client → API (stateless) → Booking Orchestrator (saga state machine)
                                │
      ┌─────────────────────────┼───────────────────────────────┐
      ▼                         ▼                                ▼
 Inventory/Airline Adapter   Payment Service                Bookings DB (SQL, replicated)
 (hold, confirm, cancel)     (charge, refund, idempotent)   + outbox → Kafka
                                                                │
                                              Async workers: reconciliation, notifications,
                                              hold-expiry sweeper, DLQ handling
```

## 5. Deep dive — Booking contention

**The race:** two users try to book the last available seat / the same fare simultaneously.

- **Short-lived hold (reservation with TTL):** on `POST /bookings`, create a `HELD` booking and request a hold from the airline (many GDS/airline APIs support a temporary PNR/price hold). The hold has an **expiry** (e.g. 5–10 min) covering the human payment time. A **sweeper** (or lazy expiry) releases abandoned holds. → *Never hold a DB lock or the seat itself across the user's think-time.*
- **Atomic guard on our side** for anything we count locally: conditional update (`... WHERE available > 0`) or optimistic version check so only one request wins; the loser gets a clean "no longer available."
- **Airline is the arbiter:** because they own inventory, the *authoritative* contention resolution happens at their `confirm` step. Our job is to (a) not oversell optimistically, (b) surface their rejection cleanly, (c) not leave a charge stranded if they reject after we charged (→ failure handling below).
- Don't use long-held pessimistic locks (`SELECT FOR UPDATE`) across external calls — you'd serialize and risk deadlocks while waiting on a slow airline API. Prefer hold+TTL + idempotency + saga.

## 6. Deep dive — Payments

- **Idempotency is non-negotiable:** payment submit carries an idempotency key; the gateway + our `payments.idempotency_key UNIQUE` ensure a retried request (after a timeout where we don't know if it succeeded) never double-charges. On retry we **query** the prior result rather than re-charging.
- **Ordering (avoid charging for a seat you can't get):** two options, state the tradeoff:
  1. **Hold → confirm-with-airline → charge:** you only take money once the seat is secured. Safest against "charged but no seat," but risks the hold expiring during a slow confirm.
  2. **Hold → charge → confirm:** faster UX/commitment, but if `confirm` fails you must **refund** (compensating action). Common in practice with robust refund handling.
- **Auth then capture:** authorize the amount at booking, **capture** only on airline confirmation → money isn't actually taken if the seat falls through; auto-void on failure. This is often the cleanest answer.
- **PCI:** never store raw card data; tokenize via the gateway.
- Handle **payment provider failure/timeout** as an ambiguous outcome (see below) — a timeout ≠ failure.

## 7. Deep dive — Failure handling (the saga)

Model booking as a **saga** — a sequence of local steps each with a **compensation**:

```
reserve(hold)   ── compensate → release hold
authorize/charge ── compensate → void/refund
confirm(airline) ── compensate → cancel PNR + refund
persist/notify
```

- **Orchestrated saga** (explicit state machine in the Booking Orchestrator) is easiest to reason about for money flows; drive transitions off the DB state + outbox events.
- **Ambiguous outcomes (the hard part):** a timeout on `charge` or `confirm` means *you don't know* whether it succeeded. Never assume failure. Resolve by:
  - **Idempotent retry** with the same key (safe to repeat), and/or
  - **Status query** against the provider/airline ("did booking X / payment Y actually go through?"), and/or
  - Park the booking in a `PENDING_RECONCILE` state and let the reconciliation worker resolve it.
- **Retries** with exponential backoff + jitter; a **dead-letter queue** for messages that keep failing → human/ops review.
- **Circuit breakers + timeouts + bulkheads** around airline and payment calls so their slowness doesn't cascade.
- **Every state transition is durable and idempotent** (outbox pattern: DB write + event emit atomic), so a crash mid-saga resumes correctly from the last committed state — no lost or double bookings.
- **Exactly-once effect** is achieved not by magic but by **at-least-once delivery + idempotent handlers**.

## 8. Deep dive — Syncing back with aggregators / airlines

The airline/aggregator is the **system of record**; our DB is a cache of their truth. Keep them consistent:

- **Reconciliation loop:** a periodic/event-driven worker compares our `PENDING/CONFIRMED` bookings against the airline's actual PNR status and fixes drift — confirm ones that actually succeeded, refund/cancel ones the airline dropped, resolve the ambiguous-timeout cases.
- **Webhooks/notifications:** subscribe to airline schedule-change / cancellation / status events; update our booking + notify the user (flight time changed, cancelled, etc.).
- **Idempotent sync:** all sync operations keyed so replays don't double-apply.
- **Inventory freshness:** the search-time price/availability (Q2) is indicative; **re-validate at booking** against live airline data, because the cached offer may be stale. If the price moved, surface it before charging.
- **Cancellations/refunds** flow the same saga in reverse and must reconcile both our ledger and theirs.

## 9. Bottlenecks & tradeoffs
- **External dependencies dominate reliability** — airline/GDS + payment gateway are slow and occasionally lie (timeouts). Design assumes they fail; reconciliation is the safety net.
- **Hold expiry vs slow confirm** — tune TTL; extend hold if confirm is in-flight.
- **Consistency vs latency** — you accept more latency (auth/capture, confirm-before-charge) to guarantee no double-charge/oversell. State this deliberately.
- **Money correctness > throughput** — the whole design biases toward *never losing or duplicating a booking or a charge*, even at some latency cost.

**Next steps:** ledger/double-entry accounting for auditability, automated refund SLAs, per-airline adapter health SLOs, and a booking-status timeline exposed to support.
