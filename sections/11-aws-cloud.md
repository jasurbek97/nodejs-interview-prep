## 11. AWS & Cloud

### Q93. Services you must know for a Node backend role

- **EC2** — VMs.
- **Lambda** — serverless functions.
- **ECS / Fargate / EKS** — container orchestration.
- **API Gateway** — HTTP entry, rate limiting, auth.
- **S3** — blob storage.
- **CloudFront** — CDN.
- **RDS / Aurora** — managed SQL.
- **DynamoDB** — managed key-value/document.
- **ElastiCache** — Redis/Memcached.
- **SQS** — queue.
- **SNS** — pub/sub.
- **EventBridge** — event bus with routing.
- **Step Functions** — workflow orchestration.
- **Secrets Manager / SSM Parameter Store** — secrets.
- **IAM** — permissions.
- **CloudWatch** — logs/metrics/alarms.
- **VPC** — networking.

### Q94. SQS vs SNS vs EventBridge

- **SQS** — a queue. One consumer per message. Use for work distribution with back-pressure.
- **SNS** — pub/sub. Fan-out to many subscribers.
- **EventBridge** — smart event bus with content-based routing and schema registry; best for decoupled microservices.

Common pattern: **SNS → SQS fan-out** (each subscriber gets a durable queue).

### Q95. Lambda — key concepts

- **Cold start** — first invocation after deploy/idle has extra latency while the runtime boots.
- **Provisioned concurrency** — keep N sandboxes warm to avoid cold starts.
- **Execution context** is reused across invocations — put DB connections/SDK clients **outside** the handler.
- Lambda has execution time limits (15 min) and payload size limits (6 MB sync, 256 KB async).
- Layers let you share code between functions.

```js
// ✅ reused across warm invocations
const ddb = new DynamoDBClient({});

export const handler = async (event) => {
  await ddb.send(new PutItemCommand({ ... }));
};
```

### Q96. S3 — how would you upload large files?

Use **multipart upload** for >100 MB:
1. `CreateMultipartUpload` → get `UploadId`.
2. Upload parts in parallel with `UploadPart` (5 MB–5 GB each, max 10 000 parts).
3. `CompleteMultipartUpload`.

For uploads from a browser without exposing credentials: server issues a **presigned URL**, client uploads directly to S3.

### Q97. DynamoDB design — why is it different?

DynamoDB is schemaless key-value. You design **around access patterns** up front, not ER diagrams.

Key concepts:
- **Partition key** — hashed to distribute data; must be high-cardinality.
- **Sort key** — ranges within a partition.
- **Single-table design** — store multiple entity types in one table, distinguished by composite keys (`USER#123`, `ORDER#...`).
- **GSI** — alternative access pattern.

Hot partitions are the main pitfall — avoid low-cardinality partition keys.

### Q98. IAM best practices

- **Least privilege** — grant the minimum permissions needed.
- Use **roles**, not long-lived access keys, wherever possible (EC2 instance profile, Lambda role, IRSA for EKS).
- **Separate accounts** per environment (Organizations).
- Avoid `*` in actions. Use condition keys (`aws:SourceIp`, `aws:ResourceTag`).
- Rotate credentials; audit with CloudTrail.

### Q99. How would you store secrets?

- **Secrets Manager** (rotation, cross-region replication) for DB credentials, API keys.
- **SSM Parameter Store** — cheaper, no rotation.
- In the app, fetch at boot or via SDK; cache in memory.
- Never in env vars baked into images, never in git, never in logs.

---

