## 20. Deep Dive — Distributed Systems & Kafka

### Q171. Kafka architecture in one picture

```
Producers ──► [ Topic A ]                        ┌─► Consumer Group X (lag tracked per partition)
              ├ Partition 0 ──► Broker 1 (leader), Broker 2 (follower) ──┤
              ├ Partition 1 ──► Broker 2 (leader), Broker 3 (follower)   ├─► Consumer Group Y
              └ Partition 2 ──► Broker 3 (leader), Broker 1 (follower) ──┘
              
KRaft / ZooKeeper ── stores metadata (which broker leads which partition, ACLs, configs)
```

Key facts:
- A **topic** is a log split into **partitions**. Order is guaranteed **per partition only**.
- Each partition has one **leader** and N-1 **followers**. Producers and consumers always talk to the leader.
- The **In-Sync Replica (ISR)** set = followers that are caught up. Leader election picks from ISR.
- The **high watermark** = the latest offset replicated to all ISRs. Consumers can only read up to the HWM.
- A partition is one **log directory** of segment files (`.log`, `.index`, `.timeindex`). Old segments are deleted by retention or compaction.
- **KRaft** (Kafka Raft) replaces ZooKeeper from 3.3+. ZK is removed entirely in 4.0.

### Q172. Producer guarantees — acks, idempotence, transactions

```js
const producer = kafka.producer({
  idempotent: true,        // dedup by (producer-id, sequence) per partition
  maxInFlightRequests: 5,  // safe with idempotence
  acks: -1,                // 'all' — wait for ISR to ack
});
```

- `acks=0` — fire and forget. Loss likely.
- `acks=1` — leader ack only. Loss if leader dies before followers replicate.
- `acks=all` (-1) — leader waits for ISR. Combined with `min.insync.replicas=2` (broker-side), this is the durable setting.
- `enable.idempotence=true` — producer assigns sequence numbers; broker drops duplicates from the same producer ID for that partition. Solves the "retry causes duplicate" problem.
- **Transactions** — write to multiple topics/partitions atomically. Combined with `read_committed` consumers, gives exactly-once across topics. Used by Kafka Streams.

### Q173. Consumer groups and rebalancing

A **consumer group** is a set of consumers that split partitions among themselves. Each partition is consumed by exactly one consumer in the group. If you have 6 partitions and 3 consumers, each gets 2; add a 4th and rebalance redistributes.

**Rebalance protocols:**
- **Eager (range / round-robin)** — everyone stops, partitions reassigned, everyone resumes. Stop-the-world for the whole group.
- **Cooperative sticky** (default in modern clients) — only revokes the partitions that need to move. Much smoother during rolling deploys.
- **Static membership** (`group.instance.id`) — consumer keeps its assignment across restarts within `session.timeout.ms`. Avoids rebalance on every deploy.

Offsets are stored in the internal `__consumer_offsets` topic. Commit them only after processing succeeds (or use exactly-once with the producer below).

### Q174. Exactly-once semantics (EOS)

Three pieces:
1. **Idempotent producer** — no duplicates per partition due to retry.
2. **Transactions** — atomic write across multiple partitions/topics.
3. **`isolation.level=read_committed`** consumer — skips messages from aborted transactions.

The classic EOS pattern (consume → process → produce) is implemented via:

```js
await producer.transaction(async (txn) => {
  await txn.send({ topic: 'out', messages: [...] });
  await txn.sendOffsets({                  // commits input offset INSIDE the txn
    consumerGroupId: 'my-group',
    topics: [{ topic: 'in', partitions: [{ partition: 0, offset: '12345' }] }],
  });
});
```

This makes "consume offset N from `in` AND publish to `out`" atomic. Outside Kafka (e.g. writing to Postgres), you need the **outbox pattern** instead — Kafka EOS doesn't extend to external systems.

### Q175. Partition key strategy & hot partitions

The producer picks a partition by `hash(key) % numPartitions` (default partitioner). Implications:

- **Same key → same partition → ordered** (good for "all events for user X are processed in order").
- **Skewed keys → hot partition** — one partition does 80% of the work, one consumer is overloaded while others idle.
- **Repartitioning is painful** — adding partitions changes the hash mapping, so old keys land in different partitions. Plan headroom upfront.

Mitigations: composite keys (`userId:bucket(0..9)`), sticky partitioner for keyless messages, monitor `BytesInPerSec` per partition.

Number of partitions = max parallelism per consumer group. Rule of thumb: 2–4× peak consumer count, capped by per-broker partition limits (~4000 per broker).

### Q176. Schema Registry & schema evolution

Confluent Schema Registry stores Avro/Protobuf/JSON schemas keyed by subject (usually `<topic>-value`). Producers serialize with the schema ID prefixed; consumers fetch the schema by ID and deserialize.

**Compatibility modes** (per subject):
- `BACKWARD` (default) — new schema can read data written by old. Safe to deploy new consumers first.
- `FORWARD` — old schema can read new data. Deploy new producers first.
- `FULL` — both.
- `NONE` — wild west.

Rules of thumb: **add fields with defaults**, never remove or rename, never change types. Use Avro unions (`["null","string"]`) for nullables.

### Q177. Dead letter topics & retry strategy

Don't loop on a poison pill — you'll block the partition forever. Pattern (popularized by Uber):

```
main_topic ──fail──► retry_5s ──fail──► retry_30s ──fail──► retry_5m ──fail──► dlq
```

Each retry topic has its own consumer with a delay (sleep, or a dedicated scheduler). The DLQ is alarmed on, hand-inspected, and replayed manually. For lower-volume use cases, a single `dlq` + a "redrive" job is enough.

### Q178. Kafka vs RabbitMQ vs SQS vs Kinesis

| | Kafka | RabbitMQ | SQS | Kinesis |
|---|---|---|---|---|
| Model | Distributed log (consumers control offset) | Queue + exchange routing | Queue (msg deleted on ack) | Distributed log |
| Ordering | Per partition | Per queue (single consumer) | FIFO queues only | Per shard |
| Throughput | Very high (MB/s per partition) | High | High (auto-scales) | High |
| Replay | Yes (retention) | No | No | Yes (24h–365d) |
| Consumers | Pull | Push | Pull (long poll) | Pull (KCL) |
| Routing | Topics + keys | Exchanges, headers, RPC | Topics via SNS fanout | None |
| Ops | Heavy (or use MSK / Confluent Cloud) | Medium | Zero | Medium |
| Sweet spot | Event streaming, CDC, audit | Task queues, RPC, complex routing | Decoupled microservices on AWS | AWS-native streaming |

**Picking:** need to replay history or fan out to many independent consumers → Kafka/Kinesis. Need flexible routing or RPC → Rabbit. Don't want to run anything → SQS/SNS.

### Q179. Outbox pattern with Kafka

Problem: writing to your DB **and** publishing to Kafka can't be atomic across systems.

Solution: write the event to an `outbox` table **in the same transaction** as the business change. A relay reads the outbox and publishes to Kafka. With **Debezium** or a custom poller doing CDC on the outbox table, the publish is decoupled from the request path.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
INSERT INTO outbox(aggregate_id, type, payload) VALUES (1, 'MoneyDebited', '{...}');
COMMIT;
-- Debezium streams the outbox INSERT to Kafka, dedup'd by event id
```

Consumers must be idempotent (the relay can retry, so duplicates happen).

### Q180. Saga pattern — orchestration vs choreography

A **saga** is a sequence of local transactions across services. If step N fails, prior steps are undone via **compensating transactions** (no global rollback exists in distributed systems).

- **Choreography** — each service emits events; others react. No central coordinator. Loose coupling but the business flow is hard to see.
- **Orchestration** — a coordinator (e.g., Temporal, AWS Step Functions, a Saga service) explicitly drives the steps. Easier to reason about and observe; the coordinator is a new component to operate.

Choose orchestration when the saga has > 3 steps or branching logic. Choose choreography for simple two-service flows.

### Q181. Two-phase commit — why it's avoided

2PC has a **prepare** then **commit** phase coordinated by a transaction manager. It works on paper but in distributed systems:

- **Blocking on coordinator failure** — if the coordinator dies after prepare, participants are stuck holding locks until it returns.
- **Latency** — every participant pays multiple network round-trips.
- **Heterogeneous systems** — most modern stores (Kafka, DynamoDB, Mongo) don't speak XA.
- **Reduces availability** — one slow participant blocks everyone.

Use sagas + idempotency + outbox instead.

### Q182. Consensus — Raft in one picture

Raft elects a leader; the leader appends entries to its log and replicates to followers. An entry is **committed** once a majority has acknowledged. Followers apply committed entries to their state machine.

```
[Client] ──► Leader ──append──► Followers ──ack──► Leader ──commit──► state machine
```

Leadership requires a quorum. With 5 nodes, you can lose 2 and still make progress. Used by etcd, CockroachDB, Consul, KRaft. **Paxos** is the older cousin — Raft is just easier to implement and explain. **ZAB** (ZooKeeper) is similar in spirit.

### Q183. Idempotency keys at the API boundary

Client sends `Idempotency-Key: <uuid>`. Server stores `(key, response)` for a TTL (24h is common). Replays return the cached response.

```
First call:  POST /payments + key=abc-123  → 201, store (abc-123, response)
Retry:       POST /payments + key=abc-123  → 201, return stored response
```

Implementation traps:
- Hash the request body and reject if the same key arrives with a different body (clients re-using keys).
- Lock per key during the first call to prevent the "two concurrent firsts" race (e.g., `INSERT ... ON CONFLICT DO NOTHING`, or Redis `SETNX`).
- Cache only **terminal** responses (2xx/4xx). Don't cache 5xx — let the client retry.

Stripe's API is the canonical example.

### Q184. CAP and PACELC

**CAP** — under a network **P**artition, choose **C**onsistency or **A**vailability. (You can't ditch P; partitions happen.)

**PACELC** extends it: **i**f **P**artitioned, choose A or C; **e**lse (normal operation), choose **L**atency or **C**onsistency.

Examples:
- DynamoDB, Cassandra: **PA / EL** — prioritize availability and low latency, eventual consistency.
- Spanner, CockroachDB: **PC / EC** — strong consistency, accept higher latency.
- MongoDB (default): **PA / EC**.

This framework matters because "we want strong consistency" usually also means "we accept higher p99 latency" — make that explicit.

### Q185. Leader election in distributed systems

Options:
- **Single-leader systems** (Kafka, Postgres): the cluster's metadata service (KRaft, Patroni) elects.
- **etcd / Consul / ZooKeeper** lease — `PUT /lock` with TTL; whoever holds it leads. Renew before expiry.
- **Database advisory lock** — `SELECT pg_try_advisory_lock(...)` in a singleton goroutine. Simple, works for small clusters.
- **Bully algorithm / Raft** — for self-contained systems.

Always include a **fencing token** (monotonic ID from the lock service) and have downstream services reject requests with stale tokens — prevents "zombie leader" double-writes.

---

