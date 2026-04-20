## 21. Deep Dive — System Design Case Studies

For every case study: **clarify** (functional + non-functional reqs, scale numbers) → **back-of-the-envelope** (QPS, storage) → **API + data model** → **architecture** → **bottlenecks + scale-out** → **failure modes**. Talk through tradeoffs, don't just present a final picture.

### Q186. Design a distributed rate limiter

**Reqs:** 100 req/s per API key, multi-region, sub-millisecond decision.

Algorithm choices:
- **Fixed window** — `INCR key:userId:minute, EXPIRE 60`. Easy, but bursts at boundaries (200 req in 1 sec across the boundary).
- **Sliding window log** — store every request timestamp. Accurate, expensive memory.
- **Sliding window counter** — weighted average of current + previous window. Good balance.
- **Token bucket** — `tokens = min(capacity, tokens + (now - last_refill) * rate); if tokens >= cost, deduct`. Handles bursts naturally.

**Architecture:**
```
Client ─► API Gateway / Envoy ─► Redis cluster (token bucket per key)
                              ─► Local in-process LRU cache (allow N before consulting Redis)
```

Use Redis with a **Lua script** so check + decrement is atomic. For multi-region, run a Redis cluster per region and accept that limits are per-region (or use CRDT counters / Cloudflare-style approximate global counters).

**Failure mode:** Redis down → fail open (allow) for availability, fail closed for security-critical limits. Always have a circuit breaker.

### Q187. Design a real-time chat system (WhatsApp-scale)

**Reqs:** 1B users, 100B msgs/day, deliver in <1s, offline delivery, group chats up to 1000 members.

**Connection layer:**
- Persistent **WebSocket** connections terminated at edge gateways. ~1M concurrent per box with C/Go (Node ~200k).
- Sticky sessions or a **presence registry** (Redis) mapping `userId → gatewayId`.

**Data model:**
- Messages partitioned by `chatId` in Cassandra/ScyllaDB (timeseries-friendly, append-heavy).
- Per-user **inbox** (a queue of message refs) for offline delivery.
- Read receipts as separate writes — don't update the message row.

**Send flow:**
1. Sender posts to the gateway → fan-out service.
2. Fan-out service writes the message (single source of truth), then for each recipient: look up gateway → push if online; else queue in their inbox.
3. Recipient ack updates the read receipts table.

**Group chats** — fan-out-on-write is fine up to ~1000; for larger groups (channels), switch to **fan-out-on-read** (recipient pulls).

**End-to-end encryption** (WhatsApp's Signal protocol) — server only sees ciphertext + routing info; key exchange is per-pair. Forward secrecy via ratcheting.

### Q188. Design a payment system / ledger

**Hard reqs:** never lose a payment, never double-charge, full auditability.

**Data model — double-entry ledger:**
```
entries(id, account_id, amount, currency, transaction_id, posted_at)
-- every transaction inserts two+ entries that sum to zero
balances = SUM(entries.amount) per account_id  -- materialized for speed
```

Never UPDATE entries — they're immutable. A "refund" is a new transaction that reverses the original.

**Transactions:** wrap entry inserts in a single DB transaction with **SERIALIZABLE** isolation, or use balance check + `UPDATE … WHERE balance >= amount` (optimistic).

**Idempotency:** every API call carries `Idempotency-Key`; the (key, transaction_id) is unique. Replays return the original transaction.

**External charges (Stripe etc.):** outbox pattern — write a `pending_charge` row and an outbox event in one tx; a worker calls Stripe and updates status. Stripe webhook reconciles.

**Reconciliation:** nightly job sums entries → must equal sum of balances. Any drift = page oncall.

**Why not eventual consistency for balances?** Overdrafts are unrecoverable; show "pending" to the user but never let two debits both pass the balance check.

### Q189. Design a notification service

**Channels:** push (APNs/FCM), email (SES/SendGrid), SMS (Twilio), in-app.

**Architecture:**
```
Producer services ─► Kafka topic: notifications
                                ↓
                        Notification Router
                  ┌─────────────┼─────────────┐
                  ▼             ▼             ▼
              Push worker  Email worker  SMS worker
                  ↓             ↓             ▼
                APNs          SES         Twilio
```

**Components:**
- **Preferences store** — per-user channel opt-in/out, quiet hours, locale.
- **Templating** — store templates by `(template_id, locale, channel)`; render at send time.
- **Throttling** — per-user (don't spam) and per-tenant (fairness).
- **Deduplication** — `(userId, eventId)` key with TTL prevents duplicate pushes if upstream retries.
- **Delivery tracking** — store `sent`, `delivered`, `clicked`, `bounced` from provider webhooks.

**Failure handling:** providers fail. Retry with backoff per channel; after N retries → DLQ → manual triage. Bounce/complaint webhooks must auto-disable a destination to keep sender reputation healthy.

### Q190. Design a search system (e.g. product catalog)

**Stack:** OpenSearch / Elasticsearch + a sync pipeline from your source-of-truth DB.

**Sync:**
- **CDC** (Debezium → Kafka → indexer) — near real-time, handles deletes correctly.
- Or batch reindex nightly + delta updates.

**Indexing:**
- Separate **read index** and **build index**; flip an alias atomically when reindexing.
- Multi-field mappings: `name.raw` (keyword) for sorting, `name.text` (analyzed) for search, `name.ngram` for typo tolerance.
- Use **synonyms** and **stemmers** per locale.

**Query:**
- `multi_match` across name/description with field boosts.
- **Function score** for popularity boost (`log1p(sales) + recency_decay`).
- Filters (category, price) as `filter` clauses (cached, no scoring).

**Ranking iteration:** log queries + clicks; train a learning-to-rank model offline; serve via OpenSearch LTR plugin.

**Pitfalls:** mapping changes require reindex; field explosion (one field per dynamic attribute) blows memory; deep pagination is slow — use `search_after`.

### Q191. Design a feed / timeline

Two strategies:
- **Pull (fan-out-on-read)** — at read time, query top-N posts from each followed account, merge in memory. Good for users with 10k followees, bad for celebrities (you do the work at every read).
- **Push (fan-out-on-write)** — when user posts, write a row to each follower's timeline. Reads are O(1). Bad for celebrities (write 100M rows per post).
- **Hybrid (Twitter):** push for normal users, pull for celebrities, merge at read. Caches the merged timeline.

**Storage:** timeline = Redis sorted set per user, capped at last ~1000 entries. Cold pages → DB.

**Ranking:** chronological is easy. ML ranking adds a feature store, model serving, and an A/B framework.

**Edge cases:** unfollow must purge entries; deleted post must invalidate caches; new follow must backfill.

### Q192. Design a URL shortener at scale

**API:** `POST /shorten {url}` → `{short: "abc12"}`; `GET /abc12` → 301 redirect.

**ID generation:** counter → base62 encode. Options:
- Single Postgres sequence (simple, central).
- Snowflake-style 64-bit ID (timestamp + machine + counter).
- Hash(url) — collisions need probing.

**Storage:** KV store (DynamoDB / Redis) keyed by short code → URL + metadata. Reads dominate writes 100:1.

**Cache:** CloudFront / CDN edge caches the redirect (`Cache-Control: public, max-age=86400`). Origin only sees cache misses + writes.

**Analytics:** redirect handler emits an event (Kafka/Kinesis); aggregated downstream. Don't synchronously write to a clicks table — it's the slowest part of the request.

**Custom URLs / spam:** uniqueness check (`INSERT ... ON CONFLICT`), bloom filter for negative cache, abuse pipeline scanning incoming URLs against threat feeds.

### Q193. Design a file upload service (S3 multipart-style)

**Direct-to-S3 (preferred):**
1. Client requests `POST /uploads` → server returns a **presigned multipart upload URL** + upload ID.
2. Client splits file into chunks (≥5MB), uploads each part directly to S3 with the presigned URL. Resumable: re-upload only failed parts.
3. Client calls `POST /uploads/:id/complete` → server calls S3 `CompleteMultipartUpload`.

Server never proxies bytes — saves bandwidth and CPU.

**Server-mediated** (legacy / non-S3): write chunks to disk; reassemble on completion; checksum (`Content-MD5` per part); virus scan via Lambda/ClamAV.

**Considerations:** lifecycle rule to abort incomplete multipart uploads after N days (otherwise you pay for orphaned parts), per-user quota, MIME sniffing on the server (not the `Content-Type` header).

### Q194. Design a job scheduler (cron at scale)

**Naive:** `cron` on one box. Single point of failure.

**Distributed:**
- **Schedule store** (Postgres): `(job_id, schedule_cron, next_run_at, owner)`.
- A worker pool polls `WHERE next_run_at <= now() FOR UPDATE SKIP LOCKED LIMIT N`, executes, updates `next_run_at`.
- Use advisory lock per job to prevent concurrent runs.
- For very high job counts, partition by `hash(job_id)` and assign partitions via consistent hashing.

**Higher-level options:**
- **Temporal / Cadence** — durable workflows with retries, signals, sagas. Default for new systems with non-trivial scheduling.
- **AWS EventBridge Scheduler** — managed, scales to millions of schedules.
- **Quartz (JVM)**, **graphile-worker** (Node + Postgres), **Sidekiq** (Ruby + Redis).

**Hard parts:** missed runs after downtime (catch up vs skip?), timezone & DST, jobs that overlap their own runtime, observability (a job failing silently is worse than a normal request failure).

### Q195. Design an audit log

**Reqs:** tamper-evident, queryable, retained 7 years, rare reads.

**Write path:**
- Every mutating action emits an event: `(actor, action, resource, before, after, ts, request_id)`.
- Events go to an **append-only** store. Immutable schema; never UPDATE/DELETE.
- For tamper-evidence, hash-chain entries: `hash_n = sha256(hash_{n-1} || event_n)`. Periodically anchor the chain hash externally (S3 Object Lock, blockchain, signed by HSM).

**Storage:** S3 in Parquet (cheap, cold, scannable via Athena). Hot index in OpenSearch for recent ~30 days.

**Reads:** rare — admin and compliance queries. Optimize for write throughput, not read latency.

**Retention:** S3 lifecycle policies + Object Lock (compliance mode) for legal hold.

**Don't:** log secrets / PII without redaction. Don't make the audit write block the user request — async via Kafka with an SLA on lag (alert if > 1 min).

---

