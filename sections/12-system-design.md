## 12. System Design & Architecture

### Q100. Monolith vs Microservices

**Monolith pros:** simpler ops, simpler transactions, lower latency between modules, easier local dev.
**Microservices pros:** independent deploys, scale critical services separately, team autonomy, fault isolation.
**Cost:** network complexity, data consistency, observability, deployment infrastructure.

Rule: start with a **modular monolith**. Split services when team/scale friction demands it.

### Q101. Synchronous vs asynchronous communication

- **Sync (HTTP, gRPC)** — simple, but couples availability.
- **Async (queue, event bus)** — decoupled, retriable, but eventual consistency.

Rule: **orchestration for synchronous workflows, choreography + events for decoupled systems.**

### Q102. Message queues — why use them?

- Decouple producers and consumers.
- Handle traffic spikes (buffering).
- Retries with backoff and DLQ (dead-letter queue).
- Enable async workflows (email, thumbnails, reports).

```
Web App → SQS → Worker → DB + S3
```

### Q103. At-least-once vs exactly-once delivery

Most queues give **at-least-once**. Design your consumers to be **idempotent** — safe to process the same message twice. Use a unique message ID + dedupe store (Redis set, DB constraint).

### Q104. Caching strategies

- **Cache-aside (lazy loading)** — app checks cache, then DB on miss, then writes back.
- **Read-through** — cache sits in front of DB transparently.
- **Write-through** — write to cache and DB simultaneously.
- **Write-behind** — write to cache, flush to DB async.

Pitfalls: stale data, cache stampede, thundering herd. Mitigations: short TTLs, jittered TTLs, request coalescing (single-flight).

### Q105. Redis — use cases

- Session store
- Rate limiter (atomic `INCR` + `EXPIRE`)
- Distributed lock (`SET NX PX`; for correctness, use Redlock)
- Pub/sub
- Queues (`LPUSH`/`BRPOP`, or Streams)
- Leaderboards (sorted sets)
- Geospatial queries
- Caching

### Q106. Horizontal vs vertical scaling

- **Vertical** — bigger box. Cheap until it isn't; hard limit.
- **Horizontal** — more boxes. Requires stateless design, load balancing, shared data layer.

### Q107. Load balancer layers

- **L4 (TCP)** — fast, port-based (AWS NLB).
- **L7 (HTTP)** — can route by path/header/cookie, TLS termination, WAF (AWS ALB, NGINX).

Session stickiness only when needed (WebSockets); otherwise prefer stateless + external session store.

### Q108. Database scaling

- **Vertical** — bigger instance.
- **Read replicas** — route reads to replicas, writes to primary.
- **Partitioning** — split tables by range/hash (same DB).
- **Sharding** — split across DBs.
- **CQRS** — separate read model (denormalized, indexed for queries) from write model.

### Q109. Eventual consistency — how do you deal with it?

- Show optimistic UI updates and reconcile.
- Use **event sourcing** to rebuild views.
- Expose **idempotent operations** and retries.
- Use **sagas** or the **outbox pattern** for distributed transactions.

### Q110. Outbox pattern

When you need to both write to the DB and publish an event without losing either:

1. In the same DB transaction, insert the event into an `outbox` table.
2. A separate poller reads `outbox`, publishes to the broker, marks as sent.

This avoids the dual-write problem (DB commits but broker publish fails, or vice versa).

### Q111. Circuit breaker pattern

Wrap remote calls with a breaker that trips (fails fast) after repeated failures. States: **closed → open → half-open**. Prevents cascading failures and gives the downstream service time to recover. Libraries: `opossum`.

### Q112. Design Twitter (high level)

- **Write path**: tweet → append to user's timeline → fan-out on write to a limited follower set. For celebrities (> N followers) use fan-out on read.
- **Timeline**: precomputed in Redis; merged with recent live tweets.
- **Storage**: Cassandra/Dynamo for tweets, sharded by user ID.
- **Search**: Elasticsearch.
- **Media**: S3 + CDN.
- **Rate limits**: Redis-backed token bucket per user.

### Q113. Design a URL shortener

- Generate short keys with base62 over an auto-increment ID, or use a hash (collision handling needed).
- Store in key-value DB (Redis/Dynamo) for O(1) lookup.
- Edge-cache popular URLs in CDN.
- Analytics via async queue → data warehouse.
- Expirations via TTL.

---

