## 8. Databases — SQL & NoSQL

### Q69. SQL vs NoSQL — when to choose what?

**SQL (PostgreSQL, MySQL)**
- Structured data, well-defined relations
- Strong consistency, ACID
- Complex queries, joins, aggregations
- Financial data, transactional systems

**NoSQL (MongoDB, DynamoDB, Cassandra)**
- Flexible/evolving schema
- Horizontal scale, massive datasets
- Document model matches app (nested, denormalized)
- Analytics, caching, real-time feeds, event logs

Reality: most backends end up using **both** — Postgres as source of truth, Redis for caching, Mongo/Dynamo for specific workloads.

### Q70. ACID properties

- **Atomicity** — all or nothing.
- **Consistency** — DB moves from one valid state to another (constraints hold).
- **Isolation** — concurrent transactions don't see each other's intermediate state.
- **Durability** — once committed, it's persisted (survives crashes).

### Q71. CAP theorem

Under a network partition (P), you must choose between **Consistency** and **Availability**:
- **CP** systems (MongoDB default, HBase) — refuse requests rather than return stale data.
- **AP** systems (Cassandra, DynamoDB) — return possibly-stale data; reconcile later.

In practice you also think about **PACELC** — in the absence of partition, tradeoff is Latency vs Consistency.

### Q72. Transaction isolation levels

From weakest to strongest:
- **Read Uncommitted** — dirty reads allowed (rare).
- **Read Committed** — Postgres default. No dirty reads; non-repeatable reads possible.
- **Repeatable Read** — MySQL default. Snapshot per transaction.
- **Serializable** — as if transactions ran one after another.

Phenomena:
- Dirty read: read uncommitted data.
- Non-repeatable read: same row has different values across reads.
- Phantom read: same range query returns new rows.

### Q73. Indexes — how do they work?

An index is a secondary data structure (usually a **B-tree**) that lets the DB find rows by key without a full scan. Tradeoffs: faster reads, slower writes, more disk space.

Types:
- **B-tree** — default, good for equality and range.
- **Hash** — equality only, fast.
- **GIN** (Postgres) — full-text, JSONB, arrays.
- **BRIN** — for huge, naturally-ordered tables (append-only logs).
- **Partial** — index only rows matching a predicate.
- **Composite** — multi-column; order matters. `(a, b)` helps `WHERE a = ?` and `WHERE a = ? AND b = ?`, **not** `WHERE b = ?`.

### Q74. What is the N+1 query problem?

Fetching a list, then firing one query per item to get relations:

```js
// N+1
const posts = await db.query('SELECT * FROM posts');
for (const p of posts) {
  p.author = await db.query('SELECT * FROM users WHERE id = $1', [p.author_id]);
}
```

Fix: **JOIN**, **IN**-batching, or DataLoader.

```js
const posts   = await db.query('SELECT * FROM posts');
const ids     = [...new Set(posts.map(p => p.author_id))];
const authors = await db.query('SELECT * FROM users WHERE id = ANY($1)', [ids]);
const map     = new Map(authors.map(a => [a.id, a]));
posts.forEach(p => p.author = map.get(p.author_id));
```

### Q75. Database normalization vs denormalization

**Normalization** (1NF–3NF+) — eliminate redundancy; update in one place. Good for OLTP.
**Denormalization** — duplicate data for read performance. Good for OLAP, dashboards, NoSQL.

Pragmatic answer: normalize first, denormalize when metrics demand it.

### Q76. MongoDB — when to embed vs reference?

- **Embed** when the child: is only accessed via the parent, doesn't grow unbounded, is small.
- **Reference** when the child: is independently queryable, shared, large, or grows unbounded.

The 16 MB document limit and "prefer working set in RAM" rule shape most design choices.

### Q77. MongoDB aggregation pipeline

```js
db.orders.aggregate([
  { $match: { status: 'paid' } },
  { $group: { _id: '$customerId', total: { $sum: '$amount' }, count: { $sum: 1 } } },
  { $sort: { total: -1 } },
  { $limit: 10 },
  { $lookup: { from: 'customers', localField: '_id', foreignField: '_id', as: 'customer' } },
]);
```

Stages you should know: `$match`, `$project`, `$group`, `$sort`, `$limit`, `$unwind`, `$lookup`, `$facet`, `$addFields`.

### Q78. PostgreSQL features worth knowing

- **JSONB** — indexed JSON with full query support.
- **Array types** — first-class.
- **CTEs** and recursive CTEs (`WITH RECURSIVE`).
- **Window functions** (`ROW_NUMBER`, `RANK`, `LAG`, `LEAD`).
- **Materialized views**.
- **Row-level security**.
- **Triggers, rules**.
- **Partitioning**.
- **`EXPLAIN ANALYZE`** — the most important command for performance work.

### Q79. Replication and sharding

- **Replication** — copies of the same data on multiple nodes. Gives HA and read scaling. Types: async (risk of data loss on failover), sync (slower).
- **Sharding** — splits data by key across nodes. Gives write scaling but complicates joins and transactions.

### Q80. Connection pooling — why does it matter in Node?

Every DB connection is expensive. Node's single process would exhaust connections quickly under load. Pools (built into `pg`, `mysql2`, Mongo drivers) reuse a fixed number of connections — typically 10–20 per process. When running N replicas, multiply: `N × pool_size ≤ db_max_connections`.

For serverless (Lambda), use a **proxy** like RDS Proxy or PgBouncer to reuse pools across invocations.

---

