# 1. File Storage Service (S3 / Dropbox style)

**One-liner:** A metadata service (SQL) that tracks files, versions, and permissions, sitting in front of an object store (blob storage) that holds the actual bytes; large files are uploaded in chunks via presigned URLs straight to the blob store, with a CDN in front for reads and async workers for replication, dedup, and virus scan.

---

## 1. Clarify requirements

**Functional**
- Upload, download, delete files.
- Store/retrieve metadata (name, size, type, owner, timestamps).
- Directory/namespace organization.
- Sharing (per-user / link-based) with permissions.
- Versioning (optional but a common deep dive).
- Large files (multi-GB) with **resumable** uploads.

**Non-functional**
- High **durability** (this is the #1 property for storage — you must not lose bytes; target 11 nines conceptually via replication/erasure coding).
- High **availability** for reads.
- Low-latency download (global users → CDN/edge).
- Scales to billions of objects, PB-scale storage.
- Security: encryption at rest + in transit, access control.

**Clarifying questions to ask the interviewer:** file size distribution? read:write ratio (usually read-heavy)? do we need strong consistency on metadata or is eventual OK for sharing? global users? do we own the blob store or use S3/GCS?

## 2. Back-of-envelope

Assume 100M users, 10 uploads/user/day, avg 1 MB (skewed — most small, few huge).
- Writes: 100M × 10 = 1B uploads/day ≈ **~12K writes/sec** average, peak ~5×.
- Storage/day: 1B × 1 MB = **1 PB/day** ingest (before dedup/compression). Dedup matters.
- Reads typically 10× writes → **~120K reads/sec**; almost all should hit CDN/cache, not origin.

These numbers justify: object store (not a DB) for bytes, CDN for reads, dedup to cut storage, and horizontal metadata sharding.

## 3. API design

```
POST   /files/initiate        -> { uploadId, chunkSize, presignedUrls[] }   # start multipart
PUT    <presignedUrl>         # client uploads each chunk directly to blob store
POST   /files/complete        -> { fileId, version }                        # finalize, idempotent
GET    /files/{id}            -> metadata + presigned download URL (redirect to CDN)
GET    /files/{id}/download   -> 302 to CDN/blob URL
DELETE /files/{id}
GET    /files?path=/foo       -> list (paginated: cursor + limit)
POST   /files/{id}/share      -> { shareLink | grants[] }
GET    /files/{id}/versions
```

Key choices:
- **Presigned URLs** so bytes flow **client → blob store directly**, never through your API tier. Your API only issues short-lived signed URLs and records metadata. This keeps the API tier stateless and cheap.
- `complete` is **idempotent** (keyed by `uploadId`) so retries after a timeout don't create duplicates.
- List is **cursor-paginated**, never offset (offset degrades on large dirs).

## 4. Data model

**Blob store (object storage):** the bytes. Objects keyed by a content-addressed or UUID key. Object stores give you cheap, durable, near-infinite scale and handle replication/erasure coding for you. This is *not* a relational DB job.

**Metadata DB (SQL, sharded):**
```
files(file_id PK, owner_id, name, path, size, content_hash, current_version, created_at, status)
file_versions(file_id, version, blob_key, size, content_hash, created_at)
blocks(block_hash PK, blob_key, refcount)        # for chunk-level dedup
permissions(file_id, grantee_id, role)           # owner/editor/viewer
shares(share_token PK, file_id, expires_at, role)
```
- Metadata is small, relational, needs transactions (rename, move, permission change) → **SQL**, sharded by `owner_id` (keeps a user's files co-located; listing a user's dir hits one shard).
- `content_hash` enables **dedup**: identical content stored once, `refcount`ed.

## 5. High-level architecture

```
                 ┌────────── CDN (reads) ──────────┐
Client ──HTTPS──> LB ──> API tier (stateless) ──> Metadata Service ──> SQL (sharded, replicated)
   │                                   │
   │                                   └──> Auth/Permission Service ──> Redis (hot metadata / sessions)
   │
   └──presigned PUT/GET──────────────> Blob / Object Store (multi-region, erasure-coded)
                                              │
                          Async workers (Kafka): dedup, virus scan, thumbnail,
                          cross-region replication, index update
```

Request flow (upload):
1. Client calls `initiate` → API authorizes, creates a pending `file` row, returns presigned chunk URLs.
2. Client `PUT`s chunks **directly to blob store** (resumable — can retry individual chunks).
3. Client calls `complete` → API verifies all chunks present, computes/records `content_hash`, flips `status=active`, emits an event.
4. Workers pick up the event: dedup, scan, replicate, index.

## 6. Deep dives (where they'll push)

### Chunking / multipart / resumable uploads
- Split large files into fixed-size chunks (e.g. 5–8 MB). Upload in parallel; retry only failed chunks; resume after disconnect using the `uploadId` + list of completed part numbers. This is what makes multi-GB uploads survive flaky mobile networks.
- Each chunk can be content-hashed → enables **block-level dedup** (Dropbox's trick: unchanged blocks in a re-uploaded file aren't re-sent).

### Deduplication
- Hash each block (e.g. SHA-256). Before storing, check `blocks` table. If present, just `refcount++` and point the version at the existing blob. Saves the ~1 PB/day problem massively for shared/duplicate content. Trade-off: hash computation cost + a hot `blocks` table (shard it by hash prefix).

### Durability & replication
- Object store replicates across ≥3 AZs, or uses **erasure coding** (e.g. Reed-Solomon 10+4) for the same durability at ~1.4× storage instead of 3×. Cross-region async replication for DR and locality.
- Metadata DB: primary + read replicas per shard; sync or semi-sync replication for the primary's durability.

### Consistency
- **Metadata:** strong within a shard (single-row/transaction ops like rename/permission are ACID). Sharing visibility can be eventually consistent.
- **Read-after-write:** after `complete`, the uploader must see the file immediately → read from primary or use write-through cache for that user's session.
- Object store writes are typically read-after-write consistent for new objects; overwrites may be eventually consistent → prefer **immutable versioned blob keys** (new version = new key) to sidestep overwrite consistency entirely.

### Downloads at scale
- Serve via **CDN** with signed URLs (short TTL). Hot objects cached at edge → origin/blob store sees a fraction of the 120K reads/sec. For private files, use signed CDN URLs so the CDN enforces expiry without hitting your API per byte.

### Security
- Encrypt in transit (TLS) and at rest (per-object keys wrapped by a KMS master key — envelope encryption). Permission checks happen at URL-issue time; presigned URLs are short-lived and scoped.

### Deletion & garbage collection
- Soft-delete metadata first (fast, reversible). A GC worker decrements `refcount`; blobs with `refcount == 0` are actually purged asynchronously. Never block the user delete on physical purge.

## 7. Bottlenecks & tradeoffs
- **Metadata DB is the scaling ceiling**, not storage (object store scales itself). Shard early by `owner_id`; watch for hot shards (a viral shared file → cache it in Redis).
- **Small-file overhead:** millions of tiny files = metadata pressure + poor object-store efficiency. Consider packing small blocks.
- **Dedup vs privacy/cost:** cross-user dedup saves storage but is a subtle info-leak/security consideration; often dedup only within a tenant.
- **Consistency vs latency:** strong metadata consistency costs cross-region latency; keep the primary regional and replicate async.

**What I'd build next:** thumbnail/preview pipeline, delta-sync (Dropbox-style) so edits transfer only changed blocks, tiered storage (hot → cold/archive by access recency).
