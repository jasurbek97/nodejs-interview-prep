## 15. DevOps, Docker & Deployment

### Q124. Dockerfile for a Node service — what does a good one look like?

```dockerfile
# Use LTS alpine for smaller images
FROM node:20-alpine AS deps
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev

FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS runtime
WORKDIR /app
ENV NODE_ENV=production
COPY --from=deps /app/node_modules ./node_modules
COPY --from=build /app/dist ./dist
COPY package.json .
USER node                       # never run as root
EXPOSE 3000
CMD ["node", "dist/server.js"]
```

Key points: **multi-stage** (smaller runtime image), **`npm ci`** (reproducible), **non-root user**, **no dev deps in runtime**.

### Q125. CI/CD pipeline — typical stages

1. Install + cache deps
2. Lint + typecheck
3. Unit tests
4. Build Docker image
5. Integration tests against real services (via Testcontainers / compose)
6. Security scan (Snyk, Trivy)
7. Push image
8. Deploy to staging, run smoke/E2E
9. Manual approval → deploy to prod (blue/green or canary)

### Q126. Zero-downtime deployment strategies

- **Rolling** — replace pods/instances N at a time.
- **Blue/green** — deploy new stack; flip LB.
- **Canary** — route X% of traffic to new version; ramp up.

Make DB migrations **backward-compatible** to survive the window where old and new code both run.

### Q127. Logging in production

- Structured JSON logs (pino).
- Correlation ID per request (propagate across services).
- Log levels by env (info in prod, debug in dev).
- Ship logs to a central store (CloudWatch, Loki, Datadog, ELK).
- Never log PII, secrets, full request bodies.

### Q128. Metrics — the 4 golden signals (Google SRE)

1. **Latency** — request duration (p50, p95, p99).
2. **Traffic** — requests/sec.
3. **Errors** — error rate.
4. **Saturation** — how "full" your system is (CPU, queue depth).

Export via Prometheus (`prom-client`), visualize in Grafana.

### Q129. Tracing with OpenTelemetry

OTel gives vendor-neutral distributed tracing. Auto-instrumentation covers HTTP, Express, Nest, Postgres, Mongo, Redis. Propagate `traceparent` header across services to stitch spans.

---

