## 13. Performance & Scalability

### Q114. How do you find a memory leak in production Node?

1. Reproduce in staging or enable heap snapshots in prod.
2. Take **two heap snapshots** a few minutes apart under load.
3. Compare retained sizes; look at constructors that grew.
4. Inspect retainer paths to find the root cause.

Tools: `node --inspect`, Chrome DevTools, `clinic.js`, `heapdump`, `0x`.

### Q115. How do you diagnose high CPU?

- Attach a profiler: `node --prof` then `node --prof-process`; or `clinic flame`; or `0x`.
- Look for hot functions, regex backtracking, sync crypto, JSON.parse on huge payloads, tight loops.
- Offload CPU work to worker threads or an external service.

### Q116. Event loop lag — what is it and how do you measure it?

If a callback blocks the event loop, timers drift and I/O stalls. Measure via `perf_hooks.monitorEventLoopDelay()`.

```js
const { monitorEventLoopDelay } = require('perf_hooks');
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => console.log('p99 lag:', h.percentile(99) / 1e6, 'ms'), 1000);
```

### Q117. Common Node.js perf wins

- `JSON.stringify/parse` is expensive — avoid reparsing.
- Use streaming for large payloads.
- Batch DB queries (DataLoader).
- Cache hot data in Redis or local LRU.
- Compile regexes once, not per call.
- Avoid `try/catch` in hot paths (pre-V8 10 it disabled optimizations; still a slight cost).
- Use `undici` instead of `node-fetch` for high-throughput HTTP clients.
- Keep objects at fixed shapes — V8 can inline-cache.
- Use `Pino` over `Winston` for logging (much faster).

### Q118. Scaling a Node.js service

1. **Cluster** across CPU cores.
2. **Horizontal** scale behind a load balancer.
3. **Cache** at every layer (CDN, Redis, in-memory).
4. **Queue** slow work.
5. **Read replicas** for the DB.
6. **Drop sync I/O** (fs.readFileSync at boot only).

---

