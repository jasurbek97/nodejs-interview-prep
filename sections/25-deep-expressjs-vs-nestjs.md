## 25. Deep Dive — Express.js (vs NestJS)

Express is a minimal, unopinionated HTTP framework. Nest is an opinionated framework that uses Express (or Fastify) as the HTTP layer underneath. Many "Express questions" really test how comfortable you are with the raw building blocks Nest hides.

Each Q below answers the Express question first, then includes a **Δ vs Nest** note where the answer differs.

### Q236. How does Express actually route a request?

Express maintains an internal **router stack** — an ordered array of layers (each layer has a path matcher and a handler). When a request comes in, Express walks the stack in declaration order; for each layer that matches, it invokes the handler. If the handler calls `next()`, the walk continues; if it calls `next(err)`, Express jumps to the next **error-handling middleware** (one with arity 4: `(err, req, res, next)`).

```js
app.use(logger);                     // matches every path
app.get('/users/:id', getUser);     // matches GET /users/:id
app.use('/admin', adminRouter);     // mounts a sub-router
app.use(errorHandler);               // 4-arg, runs on next(err)
```

Routes are just specialized middleware with a method filter. `app.get('/x', h)` is sugar over a layer that matches `GET /x`.

**Δ vs Nest:** Nest builds the same router (Express or Fastify) at startup from `@Controller`/`@Get`/`@Post` decorators. You don't see the stack, but it's there. Nest's middleware/guards/interceptors/pipes/filters are layered abstractions on top of that single underlying call chain.

### Q237. What's the difference between `app.use`, `app.METHOD`, and `Router`?

- `app.use([path], fn)` — applies `fn` to every method on `path` (or all paths if omitted). Used for cross-cutting middleware.
- `app.get/post/put/...` — method-specific terminal handlers.
- `express.Router()` — a mountable, isolated mini-app. Lets you split routes across files and mount them: `app.use('/api/v1', apiRouter)`.

Routers are the closest Express has to Nest's `@Module` — a way to compose route groups. They don't provide DI or lifecycle, just routing isolation.

**Δ vs Nest:** Nest replaces both with `@Module` + `@Controller`. A controller belongs to a module; the module wires its providers. There is no manual `app.use(router)` — Nest discovers controllers via decorators at startup.

### Q238. Express middleware execution order — the gotchas

Order is **declaration order** within an app/router. A few traps:

1. **Body parsers must come before routes that read `req.body`.**
2. **Error middleware** must be declared **last**, after all routes.
3. **Async errors don't bubble automatically in Express 4.** A rejected promise in `async (req,res,next)` is silently swallowed unless you `.catch(next)` or use `express-async-errors`. Express 5 fixes this — async errors propagate to the error handler natively.
4. `app.use('/api', m)` runs `m` for `/api`, `/api/users`, etc. — prefix match, not exact.
5. Mounted routers see paths **relative** to the mount point (`req.path` = `/users`, not `/api/users`).
6. `req.url` mutates during routing (Express strips the matched mount prefix). Use `req.originalUrl` for logging.

**Δ vs Nest:** Nest enforces a strict pipeline order — **Middleware → Guards → Interceptors (before) → Pipes → Handler → Interceptors (after) → Exception Filters**. You don't choose the order; you choose which type to write. Async errors are always caught — every async handler is awaited and exceptions go through the filter chain.

### Q239. Writing async-safe Express code (the `next(err)` problem)

Native Express 4 doesn't await your handler:

```js
// BUG: rejection is unhandled, request hangs until the client times out
app.get('/x', async (req, res) => {
  const data = await db.find();
  res.json(data);
});
```

Three fixes:

```js
// 1. Wrap manually
app.get('/x', (req, res, next) => handler(req, res).catch(next));

// 2. Wrapper helper
const asyncH = fn => (req, res, next) => Promise.resolve(fn(req, res, next)).catch(next);
app.get('/x', asyncH(async (req, res) => { ... }));

// 3. Patch Express globally
import 'express-async-errors';
```

In Express **5** (production-ready) this is built-in: returning a rejected promise from a handler/middleware automatically calls `next(err)`.

**Δ vs Nest:** Nest awaits every handler internally. `throw` from a controller method goes straight to the exception filter — no wrappers needed.

### Q240. Production-grade error handling

```js
// 4-arg signature — Express recognizes this as an error handler
app.use((err, req, res, next) => {
  if (res.headersSent) return next(err);          // delegate to default handler
  const status = err.status || err.statusCode || 500;
  req.log?.error({ err, requestId: req.id }, 'request failed');
  res.status(status).json({
    error: status >= 500 ? 'Internal Server Error' : err.message,
    requestId: req.id,
  });
});

process.on('unhandledRejection', (err) => { req.log.fatal({ err }); process.exit(1); });
process.on('uncaughtException',  (err) => { req.log.fatal({ err }); process.exit(1); });
```

Rules:
- Distinguish **operational errors** (4xx/5xx you can recover from — keep running) from **programmer errors** (bugs, invariants — crash and let your supervisor restart).
- Never leak internal messages on 5xx.
- Always include a correlation ID.
- Crash on `unhandledRejection` / `uncaughtException` — recovering from these usually leaves the process in a corrupted state.

**Δ vs Nest:** Nest provides built-in `HttpException` hierarchy and a default exception filter that formats errors as JSON. You can write a custom `@Catch()` filter to override globally or per-controller. The "operational vs programmer" distinction is the same — Nest still lets bugs crash, you still need process-level handlers.

### Q241. Production-ready Express folder structure

```
src/
  config/        # env loading + validation (zod / envalid)
  loaders/       # express.ts, db.ts, logger.ts — start order
  routes/        # one router per resource
  controllers/   # request/response shape (HTTP concerns only)
  services/      # business logic (no req/res)
  repositories/  # DB access
  middlewares/   # auth, validate, rateLimit, requestId
  schemas/       # zod / joi validation schemas
  errors/        # AppError, NotFoundError, ...
  utils/
  app.ts         # builds the app, no .listen()
  server.ts      # starts http server, handles signals
```

Two principles:
- **`app.ts` doesn't call `.listen()`.** Tests can `import('./app')` and supertest it without binding a port.
- **Controllers don't talk to the DB.** They call services. Services don't know about HTTP. This is what lets you change Express → Fastify → Lambda without rewriting business logic.

**Δ vs Nest:** Nest gives you this structure by force — `@Module`, `@Controller`, `@Injectable()` services, repositories. The trade-off: less freedom, less boilerplate, harder to deviate when you actually need to.

### Q242. Validation — the Express way

Express has none built-in. Pick one:

- **zod** (current default) — TS-first, infer types from schemas.
- **joi** — popular, mature, no TS inference.
- **express-validator** — chained validators, friendly for simple cases.
- **ajv** — JSON Schema, fastest at runtime.

```ts
import { z } from 'zod';

const CreateUserSchema = z.object({
  email: z.string().email(),
  age: z.number().int().min(13),
});

const validate = (schema) => (req, res, next) => {
  const r = schema.safeParse(req.body);
  if (!r.success) return res.status(400).json({ errors: r.error.flatten() });
  req.body = r.data;
  next();
};

app.post('/users', validate(CreateUserSchema), createUser);
```

Validate at the boundary; trust your types inside.

**Δ vs Nest:** Nest has built-in `ValidationPipe` (powered by `class-validator` + `class-transformer`). You decorate DTO classes (`@IsEmail()`, `@Min(13)`) and Nest validates the incoming body against the controller's typed parameter automatically. zod-based pipes (`nestjs-zod`) are also common.

### Q243. Auth in Express — the building blocks

Two common stacks:

**Session + cookie** (server-rendered apps):
```js
import session from 'express-session';
import RedisStore from 'connect-redis';
app.use(session({
  store: new RedisStore({ client: redis }),
  secret: process.env.SESSION_SECRET,
  cookie: { httpOnly: true, secure: true, sameSite: 'lax', maxAge: 86400_000 },
}));
```

**JWT** (SPAs, mobile, microservices):
```js
import jwt from 'jsonwebtoken';
const auth = (req, res, next) => {
  const token = req.headers.authorization?.replace('Bearer ', '');
  try { req.user = jwt.verify(token, process.env.JWT_SECRET); next(); }
  catch { res.sendStatus(401); }
};
app.get('/me', auth, (req, res) => res.json(req.user));
```

For OAuth/OIDC, **passport** has 500+ strategies but is older and verbose. Modern apps usually use **openid-client** directly or delegate to a managed identity provider (Auth0, Cognito, Clerk).

**Δ vs Nest:** Nest wraps the same logic in **Guards** (`@UseGuards(JwtAuthGuard)`) backed by `@nestjs/passport`. The strategy classes are the same Passport strategies, just wired through DI. Cleaner per-route declaration; same underlying mechanics.

### Q244. Security middleware checklist

```js
import helmet from 'helmet';
import cors from 'cors';
import rateLimit from 'express-rate-limit';
import compression from 'compression';
import hpp from 'hpp';

app.disable('x-powered-by');                    // hide Express
app.set('trust proxy', 1);                      // when behind ALB/CloudFront
app.use(helmet());                              // CSP, HSTS, X-Frame-Options, ...
app.use(cors({ origin: ALLOWED_ORIGINS, credentials: true }));
app.use(express.json({ limit: '100kb' }));     // bound body size
app.use(hpp());                                 // strip duplicated query params
app.use(rateLimit({ windowMs: 60_000, max: 100, standardHeaders: true }));
app.use(compression());
```

Beyond middleware: parameterized queries (no string concat), bcrypt/argon2 for passwords, `secure` + `httpOnly` cookies, secrets from env (never committed), `npm audit` + Dependabot in CI.

**Δ vs Nest:** identical — Nest wraps the Express server, so `app.use(helmet())` works on the underlying `app.getHttpAdapter().getInstance()`. There's also `@nestjs/throttler` for rate limiting natively.

### Q245. Streaming responses & file uploads

**Streaming a large response:**
```js
import { pipeline } from 'node:stream/promises';
app.get('/export', async (req, res) => {
  res.setHeader('Content-Type', 'text/csv');
  await pipeline(db.queryStream('SELECT ...'), csvFormatter(), res);
});
```

`pipeline` propagates errors and cleans up sockets correctly — never use `.pipe()` chains in production.

**Uploads:** `multer` is the de-facto multipart parser:
```js
import multer from 'multer';
const upload = multer({ dest: '/tmp', limits: { fileSize: 10 * 1024 * 1024 } });
app.post('/upload', upload.single('file'), (req, res) => res.json({ path: req.file.path }));
```

For large files, do **direct-to-S3** with presigned URLs instead of proxying bytes through your server.

**Δ vs Nest:** Nest exposes streaming via `StreamableFile` (`return new StreamableFile(stream)`) and uses `@nestjs/platform-express` which wraps multer (`@UseInterceptors(FileInterceptor('file'))`). Same primitives, decorator-based ergonomics.

### Q246. WebSockets in Express

Express itself doesn't speak WebSocket — you upgrade the underlying `http.Server`:

```js
import { createServer } from 'node:http';
import { WebSocketServer } from 'ws';
const server = createServer(app);
const wss = new WebSocketServer({ server });
wss.on('connection', (ws, req) => { ws.on('message', (m) => ws.send(`echo: ${m}`)); });
server.listen(3000);
```

Or `socket.io` for room/broadcast/reconnect features. Behind an ALB, set the listener to allow WebSocket upgrades and the target group's health checks accordingly.

**Δ vs Nest:** Nest has first-class WebSocket gateways (`@WebSocketGateway`, `@SubscribeMessage('event')`) with adapters for `ws`, `socket.io`, or custom transports. DI works inside gateways; same WS semantics underneath.

### Q247. Testing Express APIs

```js
import request from 'supertest';
import app from '../src/app';

describe('GET /users/:id', () => {
  it('200 on existing user', async () => {
    const res = await request(app).get('/users/1');
    expect(res.status).toBe(200);
    expect(res.body).toMatchObject({ id: 1 });
  });
  it('404 on missing', async () => {
    await request(app).get('/users/9999').expect(404);
  });
});
```

`supertest` boots the app on an ephemeral port (or accepts a request handler directly), runs HTTP requests, and tears down. Combine with a real DB + transactional rollback per test for integration confidence; reserve mocks for external HTTP / SDKs.

**Δ vs Nest:** identical pattern with `Test.createTestingModule({ imports: [AppModule] }).compile()` then `app.getHttpServer()` into supertest. Nest also lets you swap providers (`.overrideProvider(MyService).useValue(mock)`) in the testing module without touching production code.

### Q248. When to choose Express vs NestJS vs Fastify

| Need | Pick |
|---|---|
| Tiny API, full control, minimal deps | **Express** |
| Large team, opinionated structure, DI, decorators, microservices boilerplate | **NestJS** |
| Maximum throughput per core, modern plugin system, schema-first validation | **Fastify** |
| Lambda-first / serverless | **Fastify or Express** (smaller cold start than Nest) |
| Existing Express app needs structure but not a rewrite | Stay on Express, adopt **InversifyJS** or **tsyringe** for DI |

Performance: Fastify > Express > Nest-on-Express. Nest can run on Fastify (`@nestjs/platform-fastify`) and recovers most of the gap, at the cost of a few incompatible plugins.

Hiring/familiarity is often the bigger factor than benchmarks. A team that knows Nest will ship faster on Nest than on a "faster" Express they have to reinvent every cross-cutting concern in.

---
