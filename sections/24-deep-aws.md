## 24. Deep Dive — AWS

### Q223. VPC networking essentials

A **VPC** is a logically isolated network in a region. Inside it:
- **Subnets** are AZ-scoped slices of the VPC's CIDR. **Public** = route to IGW. **Private** = no IGW.
- **Internet Gateway (IGW)** — egress + ingress to the internet for public subnets.
- **NAT Gateway** — egress-only internet for private subnets (so they can reach S3 / external APIs without inbound exposure). Cost trap: ~$33/month + data processing per AZ.
- **Route table** per subnet — defines where traffic goes.
- **Security Group** — stateful firewall on ENIs (instances/RDS/ALB targets). Allow rules only.
- **NACL** — stateless firewall on subnets, allow + deny. Rarely customized.
- **VPC Endpoints** — private connectivity to AWS services without traversing the internet/NAT. **Gateway endpoints** (free) for S3 and DynamoDB; **Interface endpoints** (cost per hour + data) for everything else.
- **Transit Gateway** — hub for connecting many VPCs / on-prem.
- **VPC Peering** — point-to-point, doesn't transit.

Standard layout: 3 AZs × (public + private + private-data) = 9 subnets. ALB in public, app in private, RDS in private-data.

### Q224. ECS vs EKS vs Lambda vs Fargate

| | When to use |
|---|---|
| **Lambda** | Event-driven, sporadic load, < 15-min runs, ops you don't want. Cold starts (~100ms Node, ~1s Java) are the catch. |
| **Fargate (with ECS or EKS)** | Long-running containers, you don't want to manage EC2. Pay per task-second. |
| **ECS on EC2** | Steady high load where EC2 (esp. spot) is cheaper than Fargate. AWS-only. |
| **EKS** | Kubernetes ecosystem (Helm, operators, multi-cloud portability). Heaviest ops. |

Lambda also runs containers (up to 10 GB image). Sweet spot for "Lambda for API + Fargate for stateful workers" hybrid.

### Q225. ALB vs NLB vs CloudFront

- **ALB** (L7) — HTTP/HTTPS routing, host/path rules, OIDC integration, gRPC, WebSocket, sticky sessions. Targets: instances, IPs, Lambda, ECS tasks. Don't put a static IP on it; use DNS.
- **NLB** (L4) — TCP/UDP, ultra-low latency, **static IP per AZ**, source IP preservation. Use for non-HTTP, gaming, or when you need a fixed IP for allowlisting.
- **CloudFront** — CDN. Edge cache, WAF, custom origins (ALB/S3/anything). Use for static assets, signed URLs, geographic distribution, DDoS protection (with Shield).

Common pattern: `Route53 → CloudFront → ALB → ECS service`.

### Q226. RDS vs Aurora vs DynamoDB — picking

| | RDS Postgres/MySQL | Aurora Postgres/MySQL | DynamoDB |
|---|---|---|---|
| Engine | Standard OSS | AWS-rewritten storage | NoSQL KV |
| Scaling | Vertical, read replicas | Up to 15 aurora replicas, faster failover | Effectively unlimited (partitioned) |
| Storage | 64 TB max | 128 TB, auto-grows | Unlimited |
| Failover | 60–120s (Multi-AZ) | < 30s | N/A (multi-AZ by default) |
| Cost shape | Hourly + storage | ~20% more than RDS but more I/O | Pay per RCU/WCU or per request (on-demand) |
| When | Standard relational, modest scale | Higher throughput, faster failover, serverless v2 | Massive scale, single-digit ms, well-known access patterns |

Aurora **Serverless v2** scales compute in seconds (ACUs), good for spiky workloads. **DynamoDB on-demand** pricing is great for unpredictable load, expensive at sustained high QPS — switch to provisioned + autoscaling when you know the floor.

### Q227. DynamoDB single-table design

DynamoDB rewards precomputing your access patterns. Single-table = put all entity types into one table, distinguished by composite keys.

```
PK              SK                  type      attrs
USER#42         PROFILE             User      name, email
USER#42         ORDER#101           Order     total, status
USER#42         ORDER#102           Order     total, status
ORDER#101       ITEM#A              OrderItem qty, price
ORDER#101       ITEM#B              OrderItem qty, price
```

Queries:
- "Get user + their orders" — `Query PK = USER#42` returns the profile and all orders in one call.
- "Get an order's items" — `Query PK = ORDER#101 AND begins_with(SK, "ITEM#")`.
- "All orders by status" — needs a **GSI** with `GSI1PK = STATUS#PENDING, GSI1SK = ORDER#101`.

Rules:
- **Know access patterns first**, design schema second.
- Hot partition = throttle. Use composite or randomized PKs for skew.
- Use **transactions** (TransactWriteItems) for multi-item atomicity, but it costs 2× WCU and is limited to 100 items.
- **Streams + Lambda** = built-in CDC for projections, audit, search sync.
- **GSI sparseness** — GSIs only index items that have the GSI key, so a GSI keyed on `status` only when set is automatically a "pending orders only" index.

Read Alex DeBrie's *DynamoDB Book* — this is genuinely a different mental model from SQL.

### Q228. SQS — visibility timeout, DLQ, long polling

- **Visibility timeout** — when a consumer receives a message, it's invisible to others for this duration. If the consumer doesn't `DeleteMessage` in time, the message reappears. Set it to slightly longer than your worst-case processing time.
- **Long polling** — `WaitTimeSeconds=20` reduces empty receives and cost.
- **DLQ** — after N receives, message moves to a DLQ for manual inspection. Always configure one.
- **FIFO queues** — ordered by **MessageGroupId**, exactly-once via deduplication ID (5-min window). Lower throughput than standard (300 TPS without batching).
- **Standard queues** — at-least-once, best-effort ordering, virtually unlimited throughput.

Idempotency at the consumer is mandatory — duplicates are a feature, not a bug.

### Q229. SNS fanout + filter policies

SNS is pub/sub. One topic, many subscribers (SQS, Lambda, HTTPS, email, mobile push, Kinesis).

```
Producer → SNS topic → SQS queue (orders-service)
                    → SQS queue (analytics)
                    → Lambda    (search-indexer)
```

**Filter policies** let each subscriber receive only matching messages without app-level filtering:

```json
{ "eventType": ["OrderPlaced"], "totalUsd": [{ "numeric": [">", 1000] }] }
```

This trims the fan-out at the source and reduces SQS/Lambda invocations.

### Q230. EventBridge — buses, rules, Pipes

EventBridge = SNS+ for AWS-native events.

- **Default bus** receives events from AWS services (EC2 state changes, S3 PutObject, etc.).
- **Custom buses** for your app events.
- **Rules** match events by pattern → route to targets (Lambda, SQS, Step Functions, API Destinations, etc.).
- **Schemas** (with discovery) auto-generate types for emitted events.
- **Pipes** — point-to-point, optionally filter and enrich, between sources (SQS, Kinesis, DynamoDB streams, MQ) and targets — replaces the boilerplate "Lambda that reads from SQS and forwards to X".

When to choose EventBridge over SNS: cross-account routing, schema discovery, event replay, third-party SaaS integrations, more sophisticated filter expressions. SNS is still cheaper for simple high-throughput fan-out.

### Q231. Step Functions — when to use

Step Functions execute **state machines** defined in JSON (Amazon States Language).

- **Standard workflows** — long-running (up to 1 year), exactly-once, pay per state transition. Use for sagas, business workflows, ETL.
- **Express workflows** — sub-second, at-least-once, pay per duration. Use for high-volume request processing, IoT.

Native integrations call Lambda, ECS, DynamoDB, SQS, etc. without you writing glue code. Built-in retry/catch/parallel/map. Visual editor + execution history makes orchestrated sagas observable.

Reach for it when your workflow has > 3 sequential steps, branching, or human-in-the-loop. Don't use it as a Lambda router for simple fan-out.

### Q232. KMS, Secrets Manager, Parameter Store

- **KMS** — managed keys for encryption. You don't see the key bytes; you ask KMS to encrypt/decrypt or generate data keys (envelope encryption). Audited via CloudTrail. Multi-region keys for cross-region failover.
- **Secrets Manager** — secrets store with **automatic rotation** (built-in for RDS/Aurora/DocumentDB; custom Lambda for the rest). Cross-region replication. ~$0.40/secret/month.
- **Parameter Store (SSM)** — config + secrets store. **Standard** is free; **Advanced** is paid. No rotation. Versioning included. Good for config; OK for secrets if rotation isn't required.

Rule: secrets that need rotation → Secrets Manager; everything else → Parameter Store. Both integrate with ECS/Lambda env vars (`secrets`, not `environment`, so values are decrypted at startup and not exposed in task definitions).

### Q233. IAM — identity vs resource policies

- **Identity policy** — attached to a principal (user/role/group). "What can this principal do?"
- **Resource policy** — attached to a resource (S3 bucket, KMS key, SQS queue). "Who can do what to this resource?"

Both can grant access. The effective permission = **union** for same-account access; **both required** for cross-account access (the resource policy must explicitly allow the foreign principal).

Best practices:
- **Roles, not users.** No long-lived access keys. Use OIDC federation (e.g., GitHub Actions → AWS) for CI.
- **Least privilege** — start narrow, widen on AccessDenied.
- **Permission boundaries** — cap the maximum permissions of any role created within a team.
- **SCPs** (Service Control Policies, at AWS Organizations) — guardrails the account can't escape (deny all unencrypted S3 PutObjects, deny outside approved regions).
- **IAM Access Analyzer** — flags policies that allow unintended external access.

### Q234. Multi-AZ vs multi-region

| Failure | Multi-AZ in 1 region | Multi-region |
|---|---|---|
| AZ outage | ✅ ~30s failover | ✅ |
| Region outage | ❌ | ✅ |
| RPO | 0 (sync replica) | seconds (async) |
| Cost / complexity | Low | High |

Multi-region needs a strategy:
- **Active/passive (warm standby)** — secondary region runs at low capacity; promote on failover. Route 53 health-checked failover. Easier; some downtime.
- **Active/active** — both regions serve traffic; data layer needs conflict resolution (DynamoDB Global Tables, Aurora Global, CRDTs). Hardest.
- **Pilot light** — only critical infra running in standby; rest brought up on demand. Cheapest, highest RTO.

Most companies don't actually need multi-region. AZ failover handles the common case; regional outages are rare and your customers usually accept "AWS is down" with you. Multi-region is justified by **regulatory requirements**, **financial impact of regional outage**, or **global low-latency reads**.

### Q235. Cost optimization levers

- **Compute** — Savings Plans (1y/3y commit, 30–70% off On-Demand) for steady workloads. **Spot** (60–90% off) for stateless / batch / replicated workloads. Graviton (ARM) saves another 20%.
- **Storage** — S3 Intelligent-Tiering (auto-moves cold data to cheaper tiers). Lifecycle policies to Glacier for backups. Delete incomplete multipart uploads (default-on lifecycle rule).
- **Data transfer** — the silent killer. Cross-AZ traffic is ~$0.01/GB; cross-region is ~$0.02/GB; egress to internet is $0.05–0.09/GB. Use VPC endpoints for S3/DynamoDB (free), CloudFront for egress (cheaper than direct), and **co-locate** services with their data.
- **NAT Gateway** — $33/AZ/month + per-GB processing. Use VPC endpoints for AWS calls; consider NAT instances for low traffic; or a single shared NAT in non-prod.
- **Idle resources** — unused EBS volumes, idle ELBs, old snapshots, unattached EIPs. Tag everything; run `aws-nuke` in non-prod.
- **Right-sizing** — Compute Optimizer recommends.
- **Reservations vs Savings Plans** — RIs are stricter (instance type/region locked); Savings Plans are flexible. Default to Compute Savings Plan unless you have a stable instance family.

Set a **monthly budget alert** in AWS Budgets the day you start an account.

---

