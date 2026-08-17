# Distributed Cache System Design

Here's the detailed walkthrough of each layer — for every one, the mechanism (how) and the reasoning/tradeoff (why).

## 1. API

**One line:** Keep the surface tiny — three ops — because every added method costs latency budget and consistency guarantees.

- **How.** `set(key, value, ttl?)` writes into the owning shard and (optionally) stamps an expiry. `get(key)` returns value-or-miss. `delete(key)` removes and tombstones so a lagging replica doesn't resurrect it. All three are single-key, single-shard operations — no cross-key semantics.
- **Why single-key only.** The moment you allow multi-key ops (`mget` across shards, range scans, "get all keys with prefix"), you've reintroduced coordination, fan-out latency, and partial-failure handling — the exact things a cache exists to avoid. Cross-shard atomicity would force 2-phase commit and blow the <10ms budget.
- **Why `ttl` is per-key, not global.** Different data has different staleness tolerance (session tokens = minutes, rendered HTML = seconds). Per-key TTL lets callers tune freshness vs hit-rate themselves.
- **Miss semantics.** A `get` miss is a normal, expected outcome (cold key, evicted, expired) — the client falls back to the source of record and re-populates. The cache never blocks waiting to "find" a key.

## 2. Capacity

**One line:** The design is **data-bound, not throughput-bound** — RAM size sets the node count, and QPS falls out for free.

- **Data → shards.** 1TB working set. Use nodes with ~128–256GB RAM but only budget ~50% for the keyspace (rest for OS, replication buffers, fragmentation headroom) → ~64–128GB usable each → **~16 shards** to hold 1TB with room to grow.
- **Throughput is not the constraint.** 100k RPS ÷ 16 shards ≈ **6–7k RPS per shard**. A single in-memory node does 100k+ ops/s, so you're at <10% of node capacity. This is why you *don't* add nodes for QPS here — you add them for bytes.
- **Latency budget.** In-memory lookup is sub-microsecond; the 10ms is almost entirely network. So the levers that matter are: one network hop (client-side routing, not proxy chains), connection pooling, and keeping the node single-threaded-fast (no GC pauses / lock contention).
- **Replication multiplier.** RF=3 → ~48 nodes total. That's a memory-cost decision (3× RAM spend) traded for availability — covered in §5.
- **Why leave headroom.** LRU works by staying near-full and evicting; if you size for exactly 1TB you evict constantly and hit-rate craters. You want the working set to *fit* with margin.

## 3. Single-node design

**One line:** A hashmap gives O(1) lookup; a doubly linked list gives O(1) recency ordering — together they make LRU free.

- **How the two structures combine.** The hashmap maps `key → pointer to a DLL node`. The DLL holds the actual entries ordered by recency: **head = most-recently-used, tail = least**. 
  - `get`: hashmap finds the node in O(1) → unlink it → splice to head. Now it's "fresh."
  - `set`: insert at head; if over capacity, drop the tail node (and its hashmap entry).
  - `delete`: hashmap locates node → unlink from list → erase from map.
- **Why a doubly linked list specifically.** You need O(1) *removal from the middle* (when you touch a key in the interior and promote it to head). A singly linked list can't unlink in O(1) because you can't reach the predecessor; an array can't either (shifting is O(n)). Doubly linked = both neighbors known = O(1) splice.

Here's the structure — the map indexes into the list, and every access re-threads the touched node to the head:Now for the TTL side of the same node:

- **How expiration works — two mechanisms together.** Each entry carries an absolute expiry timestamp.
  - *Lazy (passive):* on every `get`, check `now > expiry`; if so, treat as a miss and drop it. Guarantees you never return stale data, costs nothing until the key is touched.
  - *Active (background):* a sampler periodically picks a random batch of keys, deletes the expired ones, and — if the expired fraction is high — repeats immediately. This reclaims RAM held by keys that are expired but never read again.
- **Why both.** Lazy alone leaks memory (a written-once, never-read expired key sits forever). Active alone wastes CPU scanning the whole keyspace. Random-sampling active expiry is probabilistic — cheap, and self-throttling (scans harder only when there's actually a lot to reap). This is exactly Redis's approach.
- **Why absolute timestamps, not countdown timers.** A timer per key is millions of timers; a stored expiry is one comparison at access time. O(1) memory per key, no timer wheel.

## 4. Sharding — consistent hashing

**One line:** Consistent hashing lets you add/remove a node while remapping only ~1/N of keys instead of the entire keyspace.

- **Why not plain `hash(key) % N`.** Modulo hashing ties every key's location to N. Change N (add/remove a node) and *almost every* key moves → a cache-wide miss storm hammers your database at the worst moment (you're scaling *because* you're under load).
- **How the ring works.** Hash the key space onto a fixed circle (e.g. 0…2³²). Place each node at one or more points on the ring. A key is owned by the **first node encountered clockwise** from the key's hash. Add a node → it inserts at one arc and steals only the keys in that arc from its clockwise neighbor. Everyone else is untouched.

Here's the ring — keys land at their hash and walk clockwise to the next node:- **Why virtual nodes (vnodes).** With one point per node, arcs are uneven — one node randomly owns a huge arc and gets hot, and when a node dies its *entire* load dumps onto a single neighbor. Giving each physical node many ring positions (say 100–200) smooths the distribution and, critically, spreads a dead node's keys across *all* survivors instead of one.
- **How clients find the owner.** Every client (or proxy) holds the ring map. Routing is a local hash + lookup — no coordination hop, preserving the latency budget. The coordination service just pushes ring updates on membership change.

## 5. High availability — replication

**One line:** Each shard is a primary plus async replicas, so a node death costs a failover, not data unavailability.

- **How replication works.** Writes land on the shard's primary, which acknowledges the client immediately and streams the change to replicas asynchronously. Replicas can serve reads to spread load.
- **Why async, not sync.** Synchronous replication means the primary waits for replica acks before returning — that adds a full network round-trip to every write, breaking <10ms, and makes you unavailable if a replica is slow/down. Async keeps writes fast and the shard available under replica failure. The cost: a replica may lag by a few ms, so a read right after a write *might* see the old value — acceptable, since NFR1 explicitly allows eventual consistency (and it's a cache; the DB is the truth).
- **How failover works.** The coordination service heartbeats every node. On missed heartbeats it declares the primary dead, promotes the most up-to-date replica, and pushes the new ring/topology to clients. In-flight writes to the dead primary that hadn't replicated are lost — again fine for a cache.
- **Why detection needs a quorum.** A single monitor can be fooled by a network blip and wrongly promote, causing split-brain (two primaries). Requiring several coordinator nodes to *agree* a primary is down before promoting prevents flapping. This is the Redis Sentinel / cluster model.
- **Why RF=3 specifically.** RF=2 tolerates one failure but leaves you with zero redundancy mid-failure; RF=3 tolerates one failure and still has a spare during recovery. Beyond 3, you're paying more RAM for diminishing availability gains.

## 6. Hot keys / hot shards

**One line:** Consistent hashing spreads keys evenly, but it can't spread *one* viral key — so you handle skew separately.

- **The two distinct problems.**
  - *Hot key:* a single key (celebrity profile, flash-sale item) gets a disproportionate share of traffic — one node saturates while others idle.
  - *Hot shard:* one node's whole arc runs hotter than others (uneven vnode placement, or a cluster of popular keys).
- **How to detect.** Per-node request counters / top-K heavy-hitter tracking (count-min sketch) surface which keys or shards are spiking, feeding the mitigation automatically.
- **How to fix a hot key.**
  - *Replicate the key* to multiple nodes and fan reads across them — N replicas → N× read capacity for that key.
  - *Client-side micro-cache:* clients cache the hot value locally for a very short TTL (e.g. 1s), absorbing most reads before they ever hit the cluster. Cheap and extremely effective for read-heavy hot keys.
  - *Key-splitting:* store `key#1…key#k` copies and have clients pick one at random, spreading writes+reads across shards (used when the hot key is also write-heavy).
- **How to fix a hot shard.** Add dedicated read replicas to that shard, or rebalance vnodes so the hot arc is subdivided across more physical nodes.
- **Why not just shard harder.** Sharding distributes *across* keys; it does nothing for load concentrated *on* a single key — hashing sends every request for that key to the same node by definition. Skew is a separate axis and needs replication/caching, not more partitions.

**Net:** the design keeps every hot path in-memory and one hop away (latency), scales by bytes via consistent hashing (scale), survives node loss via async replicas + quorum failover (availability), and absorbs skew with per-key replication and client micro-caching (hot keys) — while deliberately punting durability, strong consistency, querying, and transactions to the system of record.