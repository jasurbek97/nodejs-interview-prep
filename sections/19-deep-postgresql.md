## 19. Deep Dive — PostgreSQL

### Q159. MVCC — how does it actually work?

Postgres uses **Multi-Version Concurrency Control**: every row has hidden columns `xmin` (the transaction that inserted it) and `xmax` (the one that deleted/updated it). UPDATE doesn't overwrite — it writes a **new tuple version** and sets `xmax` on the old one.

A transaction sees only tuples whose `xmin` committed before it started and whose `xmax` either didn't commit or is in the future, per its **snapshot**.

Consequences:
- **Readers never block writers, writers never block readers.**
- Old versions accumulate (**bloat**) until VACUUM removes them.
- Long-running transactions hold back the **xmin horizon**, preventing VACUUM from cleaning anything newer than them. This is the #1 cause of unexpected bloat.

### Q160. VACUUM, autovacuum, and bloat

**VACUUM** marks dead tuples as reusable space (does **not** return space to the OS). **VACUUM FULL** rewrites the whole table — locks it exclusively. Avoid in production; use `pg_repack` instead.

**Autovacuum** runs in the background, triggered when the dead-tuple count exceeds `autovacuum_vacuum_scale_factor * rows + threshold`. Defaults are too lazy for high-write tables — tune per-table:

```sql
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.02,
  autovacuum_analyze_scale_factor = 0.01
);
```

**HOT updates** (Heap-Only Tuple) — if no indexed column changes and there's room in the same page, the new version stays in the same block and indexes don't have to be touched. Maximize HOT by leaving fillfactor headroom: `WITH (fillfactor = 80)`.

Monitor bloat via `pg_stat_user_tables.n_dead_tup` and the `pgstattuple` extension.

### Q161. WAL, checkpoints, and replication

**WAL** (Write-Ahead Log) — every change is appended to WAL before being applied to data pages. Crash recovery replays WAL from the last checkpoint.

**Checkpoint** flushes dirty pages to disk and trims WAL. Tuned via `checkpoint_timeout` (default 5min) and `max_wal_size`. Aggressive checkpoints = small WAL but heavy I/O spikes; relaxed checkpoints = recovery is slower.

**Replication** ships WAL to replicas:
- **Streaming replication (physical)** — byte-for-byte copy of WAL. Replicas are read-only standbys, identical layout.
- **Logical replication** — decodes WAL into row-level changes (INSERT/UPDATE/DELETE) per publication. Replicas can have a different schema, version, or only a subset of tables. Used for zero-downtime upgrades and CDC pipelines.
- **Replication slots** — pin WAL on the primary until consumers ack. **Always monitor `pg_replication_slots.confirmed_flush_lsn`** — an abandoned slot will fill the disk.

### Q162. Index types — when to use which

| Type | Use case |
|---|---|
| **B-tree** (default) | Equality, ranges, ORDER BY, LIKE 'prefix%' |
| **Hash** | Equality only. Rarely better than B-tree; smaller for huge equality-only loads. |
| **GIN** | "Many values per row" — JSONB, arrays, full-text (`tsvector`), trigrams (`pg_trgm`) |
| **GiST** | Geometric, range, trigram, full-text. Lossy but flexible. |
| **SP-GiST** | Non-balanced trees (quad-trees, radix). Niche. |
| **BRIN** | Huge, naturally-ordered tables (time-series). Block-range index — tiny but coarse. |
| **Bloom** (extension) | Multi-column equality where any combination of columns may be queried |

**Partial indexes** index only a subset (`WHERE deleted_at IS NULL`) — smaller, faster.
**Expression indexes** index a function value (`CREATE INDEX ON users (lower(email))`).
**Covering indexes** include extra columns (`INCLUDE (name, email)`) for index-only scans.

### Q163. Reading EXPLAIN ANALYZE

```
Nested Loop  (cost=0.43..56.75 rows=1 width=44) (actual time=0.025..0.832 rows=12 loops=1)
  ->  Index Scan using orders_user_id_idx on orders  (...) (actual rows=12 ...)
        Index Cond: (user_id = 42)
  ->  Index Scan using users_pkey on users  (...) (actual rows=1 loops=12)
        Index Cond: (id = orders.user_id)
Planning Time: 0.140 ms
Execution Time: 0.901 ms
```

What to look for:
- **`actual rows` vs `estimated rows`** — if off by 100×, statistics are stale (`ANALYZE` the table) or correlated columns need extended stats (`CREATE STATISTICS`).
- **`Seq Scan` on a large table with a filter** — missing or unused index.
- **`loops=N`** in nested loops — the inner side runs N times. If N is huge, a `Hash Join` or `Merge Join` would beat it.
- **`Buffers: shared hit=… read=…`** (with `BUFFERS` flag) — read = disk fetch. High `read` = cold cache or table too big for shared_buffers.
- **`Sort Method: external merge Disk`** — `work_mem` is too small for the sort.

Use `EXPLAIN (ANALYZE, BUFFERS, VERBOSE, FORMAT TEXT)` and paste into [explain.dalibo.com](https://explain.dalibo.com).

### Q164. Partitioning

Native **declarative partitioning** since PG 10:

```sql
CREATE TABLE events (id bigserial, occurred_at timestamptz NOT NULL, ...)
  PARTITION BY RANGE (occurred_at);

CREATE TABLE events_2026_04 PARTITION OF events
  FOR VALUES FROM ('2026-04-01') TO ('2026-05-01');
```

Strategies:
- **RANGE** — time-series (most common).
- **LIST** — by tenant, region.
- **HASH** — even spread when no natural key.

**Wins:** drop old data with `DROP TABLE events_2025_01` (instant vs. DELETE), partition pruning makes scans target one partition, autovacuum scales per partition.

**Watch out:** every query must include the partition key to benefit from pruning. Unique constraints must include the partition key. Use `pg_partman` to automate creating/retiring partitions.

### Q165. Row-level locks and the SKIP LOCKED queue pattern

Postgres can be a job queue without Redis/Kafka:

```sql
BEGIN;
SELECT id, payload FROM jobs
  WHERE status = 'pending'
  ORDER BY created_at
  FOR UPDATE SKIP LOCKED
  LIMIT 10;
-- process the rows
UPDATE jobs SET status = 'done' WHERE id = ANY($1);
COMMIT;
```

`FOR UPDATE` takes a row lock; `SKIP LOCKED` makes other workers grab the next free row instead of blocking. Throughput scales linearly until you hit lock contention or vacuum overhead. Used in production by GitLab, Sidekiq Pro, [graphile-worker](https://github.com/graphile/worker).

### Q166. Advisory locks

App-level locks not tied to any row:

```sql
SELECT pg_try_advisory_lock(123);  -- non-blocking
SELECT pg_advisory_xact_lock(123); -- released at COMMIT/ROLLBACK
```

Use cases: cluster-wide singletons (only one cron worker runs at a time), preventing concurrent migrations, leader election in small clusters.

### Q167. JSONB — indexing and pitfalls

`JSONB` stores parsed JSON in a binary format. Indexable with GIN:

```sql
CREATE INDEX ON docs USING GIN (data);                    -- supports @> ? ?| ?&
CREATE INDEX ON docs USING GIN (data jsonb_path_ops);     -- smaller, supports only @>
CREATE INDEX ON docs ((data->>'status'));                 -- B-tree on one field
```

**When to use JSONB:** truly schemaless data, sparse attributes, denormalized read models.
**When NOT to:** anything you'll filter, sort, or aggregate by frequently — promote those to columns. JSONB columns can't have foreign keys, statistics on inner fields are weak, and updates rewrite the whole document (no partial-field bloat avoidance).

### Q168. LISTEN / NOTIFY

Lightweight pub/sub without an external broker:

```sql
NOTIFY orders, '{"id":42}';
LISTEN orders;
```

Limits: payload < 8 KB, messages dropped if no listener is connected, no persistence, no replay. Good for **cache invalidation** signals, low-volume real-time updates. For anything durable, use a queue table + LISTEN as a wake-up signal.

### Q169. PgBouncer — pooling modes that matter

| Mode | Pool reuse boundary | What breaks |
|---|---|---|
| **Session** | Connection close | Nothing (transparent) — minimal benefit |
| **Transaction** | Each COMMIT | Session features: `SET`, prepared statements (pre-PG14), `LISTEN`, advisory locks across statements |
| **Statement** | Each statement | Multi-statement transactions. Don't use. |

**Transaction mode** is the standard. Common gotchas:
- Don't `SET search_path` on the session — set it per query or via `pgbouncer.ini`.
- `LISTEN` doesn't work; use a dedicated direct connection.
- ORMs that rely on server-side prepared statements need the protocol-level prepared statement support added in PgBouncer 1.21+.

Sizing: app pool = `(transaction_pool_size / app_replicas)`, and Postgres `max_connections` ≥ `pool_size * pgbouncer_instances + reserved`.

### Q170. Common Postgres pitfalls at scale

- **`SELECT count(*) FROM big_table`** is slow (must scan to count visible tuples). Use approximations: `pg_class.reltuples` or maintain a counter table.
- **`OFFSET 100000 LIMIT 20`** — keyset pagination wins (`WHERE created_at < $cursor ORDER BY created_at DESC`).
- **`ORDER BY random() LIMIT 1`** — full scan + sort. Use `TABLESAMPLE` or precomputed random IDs.
- **Long transactions** — block VACUUM, accumulate locks, kill replication. Set `idle_in_transaction_session_timeout` and `statement_timeout`.
- **Index on every column** — write amplification. Audit with `pg_stat_user_indexes` (`idx_scan = 0` = unused).
- **DDL during traffic** — most ALTERs take an `ACCESS EXCLUSIVE` lock. Use `CREATE INDEX CONCURRENTLY`, `ALTER TABLE … ADD COLUMN … (no default in PG ≥11)`, and `lock_timeout`.
- **`NOT IN (subquery)`** — NULLs make it return wrong results. Use `NOT EXISTS`.
- **`enum` types** — adding a value is fine, removing requires recreating the type. Prefer a lookup table for anything that may evolve.
- **TOAST surprises** — large field values get stored out-of-line and (de)compressed; a slow query may be paying for TOAST decompression, not the join.

---

