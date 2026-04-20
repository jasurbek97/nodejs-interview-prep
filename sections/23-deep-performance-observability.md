## 23. Deep Dive — Performance & Observability

### Q211. The pillars of observability — and the fourth one

The classic three: **logs, metrics, traces**. The fourth often missed: **profiles** (continuous CPU/heap/lock profiling).

| Pillar | Cardinality | Cost | Best for |
|---|---|---|---|
| Metrics | Low (label combos) | Cheapest | Aggregates: rate, error rate, latency percentiles |
| Logs | Unbounded | Expensive at volume | Forensics on a single request |
| Traces | Per-request, sampled | Medium | "Where in the call chain did time go?" |
| Profiles | Continuous, low overhead | Cheap | "Which functions burn CPU/allocate?" |

You need all four to debug production issues confidently. Logs tell you *what happened*, metrics *how often*, traces *where*, profiles *why* (in CPU terms).

### Q212. RED vs USE — pick the right framework

- **RED** (services / requests) — **R**ate, **E**rrors, **D**uration. For every endpoint, dashboard those three. Originated by Tom Wilkie at Weaveworks.
- **USE** (resources) — **U**tilization, **S**aturation, **E**rrors. For every resource (CPU, memory, disk, network), dashboard those three. Brendan Gregg's framework.

Use both. RED tells you the user-visible symptom. USE tells you which resource is the cause. A service with high duration (RED) often correlates with high saturation (USE) on a downstream DB.

### Q213. SLI, SLO, error budgets

- **SLI** — what you measure (e.g., "% of requests served < 200ms").
- **SLO** — your target (e.g., "99.9% of requests < 200ms over 30 days").
- **SLA** — the contractual version (with consequences). Often looser than the SLO.
- **Error budget** — `1 - SLO`. With a 99.9% SLO, you have 43 minutes of "burn" per month.

Operational policy: when you've burned > 50% of the budget early in the window, freeze risky deploys; when you've stayed under, you're allowed to ship faster. Makes "stability vs velocity" a measurable trade-off, not an argument.

**Multi-window burn-rate alerts** beat threshold alerts: alert if you'd burn the monthly budget in a few hours at the current rate.

### Q214. Instrumenting Node with OpenTelemetry

OTel is the vendor-neutral standard. Auto-instrumentation hooks common libs (HTTP, Express, Nest, Postgres, Redis, Kafka, Mongo).

```js
// tracing.js — required BEFORE any other import
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({ url: process.env.OTEL_ENDPOINT }),
  instrumentations: [getNodeAutoInstrumentations()],
});
sdk.start();
```

Run with `node --import ./tracing.js app.js`. Add custom spans for business logic:

```js
import { trace } from '@opentelemetry/api';
const tracer = trace.getTracer('orders');
await tracer.startActiveSpan('orders.place', async (span) => {
  span.setAttribute('orders.itemCount', items.length);
  try { /* work */ } finally { span.end(); }
});
```

Send to Jaeger / Tempo / Honeycomb / Datadog via OTLP — the exporter is the only thing that changes per backend.

### Q215. Structured logging done right

Rules:
- **JSON, one line per event.**
- Always include `timestamp`, `level`, `service`, `traceId`, `spanId`, `requestId`, `userId` (when present).
- Don't log secrets, tokens, full request bodies. Have a denylist.
- Levels: `error` → page, `warn` → review, `info` → audit trail, `debug` → off in prod.
- Use a fast logger: **pino** (~5× faster than Winston, ~50µs per log).

```js
import pino from 'pino';
const log = pino({ redact: ['req.headers.authorization', 'password'] });
log.info({ userId, orderId }, 'order placed');
```

Get `traceId`/`spanId` from OTel context via AsyncLocalStorage so every log line is correlatable to a trace.

### Q216. Sampling traces

100% sampling is expensive at scale. Strategies:
- **Head-based** — decide at the root span. Cheap, fast, but you lose interesting traces (errors) at random.
- **Tail-based** — collect everything, decide at the collector after the trace completes. Lets you keep all errors and slow traces. Needs a buffer (Tempo, OTel collector with `tail_sampling` processor).

A sensible default: 100% of errors, 100% of slow (`> p95`), 1–5% of everything else.

### Q217. Continuous profiling

Profilers like **Pyroscope** / **Parca** / **Datadog Profiler** sample CPU, heap, and goroutines/asyncs continuously in production at < 1% overhead. You get "what was the system doing at this moment?" for any historical timestamp.

Killer feature: **diff** the profile from before vs after a deploy to see exactly which functions started taking more CPU. Saves hours of guessing.

In Node: `--cpu-prof` and `--heap-prof` flags emit `.cpuprofile` / `.heapprofile` files. Open in Chrome DevTools or feed to a continuous profiler.

### Q218. Benchmarking Node APIs

- **autocannon** — Node-native, low overhead. Good for HTTP.
  ```bash
  autocannon -c 100 -d 30 http://localhost:3000/api
  ```
- **k6** — JS-scripted, scenario-based load tests. Good for sustained tests with ramps.
- **wrk** — C, lowest overhead. Use when the load generator itself is the bottleneck.

Best practices:
- Run on a separate box from the SUT. Localhost benchmarks lie.
- **Warm up** for 30–60s before measuring.
- Report **p50/p95/p99 + max**, not just averages.
- Hold one variable; change one. Test against a stable baseline.
- Use the **same payload distribution** as production.

### Q219. Reading flame graphs

X-axis = time-on-CPU (sample count), **not** wall-clock time. Y-axis = stack depth (caller below callee). Width = how much CPU. Look for **plateaus** (wide flat areas) — those are your hot functions.

```
+---------------------+
|       handler       |   <- if this is wide, it's the bottleneck
+---+-----+-----+-----+
|JSON|qury| ... |     |
+---+-----+-----+-----+
```

In Node, `0x` and clinic `flame` produce these. Look at **self time** (top of stack) vs **total time** (everything called below). Differential flamegraphs (red = added CPU, blue = removed) are gold for regression hunting.

### Q220. GC tuning in Node

Default Node ships with reasonable V8 GC defaults. Tune when you have evidence (`--trace-gc`, `perf_hooks` GC events).

Common knobs:
- `--max-old-space-size=4096` — raise old-space cap when you OOM with `JavaScript heap out of memory`.
- `--max-semi-space-size=64` — bigger young space = fewer scavenges, but each is bigger.
- `--gc-interval=N` (rarely useful) — force GC every N allocations.

What usually fixes GC pressure isn't a flag, it's allocating less:
- Reuse buffers (`Buffer.allocUnsafe` from a pool) instead of `Buffer.alloc` per request.
- Stream large responses instead of buffering.
- Avoid huge object literals in hot paths.
- Watch for accidental closures keeping big graphs alive.

### Q221. HTTP/2, HTTP/3, keep-alive

- **HTTP/1.1** — one request at a time per connection (head-of-line blocking). Use `keep-alive` and per-host connection pools.
- **HTTP/2** — multiplexed streams over one TCP connection. Eliminates HoL at the HTTP layer; at the TCP layer one lost packet still stalls all streams.
- **HTTP/3** — over QUIC (UDP). No transport-level HoL; faster handshakes (0-RTT). Best for high-latency / lossy networks.

In Node, `http2` is built-in for servers. For clients, `undici` is much faster than `http` and supports HTTP/2 and connection pooling out of the box. **Always reuse an HTTP agent** for outbound calls — creating a new connection per request is the #1 hidden latency cost.

### Q222. Diagnosing event loop lag end-to-end

```js
import { monitorEventLoopDelay } from 'node:perf_hooks';
const h = monitorEventLoopDelay({ resolution: 10 });
h.enable();
setInterval(() => {
  console.log({ p50: h.percentile(50), p99: h.percentile(99), max: h.max });
  h.reset();
}, 5000);
```

If p99 > 50ms regularly, something is blocking the loop. Investigate in this order:

1. **Sync CPU work** — JSON parsing of a 5MB body, synchronous crypto, regex catastrophic backtracking. Move to `worker_threads` or stream.
2. **Massive promise micro-task floods** — `Promise.all([...10k items])` resolving at once. Process in chunks.
3. **Native addon blocking** — some addons run on the JS thread. `node-report` shows native frames.
4. **GC pauses** — `--trace-gc` will show them. Often a symptom of allocation pressure (above).
5. **libuv pool saturation** — fs/crypto queueing. `UV_THREADPOOL_SIZE`.

Export the lag as a metric (`event_loop_lag_p99_ms`) and alert on it — it's the single best leading indicator of a Node service in trouble.

---

