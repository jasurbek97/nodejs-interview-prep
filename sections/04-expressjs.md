## 4. Express.js

### Q37. What is middleware?

A function with signature `(req, res, next)` that sits in the request pipeline. It can:
- Modify `req`/`res`
- End the request (`res.send`)
- Pass control via `next()`
- Pass error via `next(err)`

```js
app.use((req, res, next) => {
  req.startTime = Date.now();
  res.on('finish', () => console.log(Date.now() - req.startTime, 'ms'));
  next();
});
```

### Q38. Error-handling middleware

An error handler has **4 arguments** — Express recognizes it by arity.

```js
app.use((err, req, res, next) => {
  req.log.error(err);
  res.status(err.status ?? 500).json({ error: err.message });
});
```

Errors from sync code are caught automatically. For **async routes**, throw via `next(err)` or use a wrapper:

```js
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

app.get('/user', asyncHandler(async (req, res) => {
  const u = await db.findUser(req.query.id);
  res.json(u);
}));
```

Express 5 handles this automatically.

### Q39. Middleware execution order

Middleware runs **in the order registered**. Mounting order matters: put body parsers before routes, put error handlers **last**.

```js
app.use(helmet());            // security headers
app.use(cors());
app.use(express.json());      // body parser
app.use('/api', apiRouter);
app.use(notFound);
app.use(errorHandler);        // last
```

### Q40. How do you structure a production Express app?

- Layered: **routes → controllers → services → repositories**.
- Validation (zod/joi) at the controller boundary.
- Centralized error handler returning RFC 7807 (`application/problem+json`).
- DI container (awilix, tsyringe) for testability.
- Request-scoped logging (pino + correlation ID).
- Graceful shutdown.

### Q41. `req.params` vs `req.query` vs `req.body`

- `req.params` — URL path variables (`/users/:id` → `req.params.id`).
- `req.query` — URL query string (`?limit=10`).
- `req.body` — request body (requires body-parser middleware).

### Q42. Common security middleware

- `helmet` — sets secure HTTP headers.
- `cors` — configures Cross-Origin Resource Sharing.
- `express-rate-limit` — throttle per-IP.
- `express-validator` / `zod` — input validation.
- `hpp` — HTTP parameter pollution protection.

---

