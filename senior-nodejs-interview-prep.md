# Senior Node.js Backend Interview — Complete Preparation Guide

> A battle-tested study guide covering everything a senior Node.js engineer is expected to know: internals, frameworks, databases, security, system design, and more. Every question includes a concise answer, code examples where relevant, and common follow-up traps.

---

## Table of Contents

1. [Node.js Core & Internals](#1-nodejs-core--internals)
2. [JavaScript & TypeScript Deep Dive](#2-javascript--typescript-deep-dive)
3. [Asynchronous Programming](#3-asynchronous-programming)
4. [Express.js](#4-expressjs)
5. [NestJS](#5-nestjs)
6. [REST API Design](#6-rest-api-design)
7. [Authentication, Authorization & Security](#7-authentication-authorization--security)
8. [Databases — SQL & NoSQL](#8-databases--sql--nosql)
9. [ORMs — Prisma, Sequelize, TypeORM](#9-orms--prisma-sequelize-typeorm)
10. [GraphQL](#10-graphql)
11. [AWS & Cloud](#11-aws--cloud)
12. [System Design & Architecture](#12-system-design--architecture)
13. [Performance & Scalability](#13-performance--scalability)
14. [Testing](#14-testing)
15. [DevOps, Docker & Deployment](#15-devops-docker--deployment)
16. [Coding Challenges](#16-coding-challenges)
17. [Behavioral / Senior-Level Questions](#17-behavioral--senior-level-questions)

---

## 1. Node.js Core & Internals

### Q1. What is Node.js and how does it work under the hood?

Node.js is a JavaScript runtime built on Chrome's **V8 engine**. It uses a **single-threaded, non-blocking, event-driven** architecture powered by **libuv** (a C library that implements the event loop and thread pool).

Key components:
- **V8** — compiles JS to machine code
- **libuv** — provides event loop + thread pool (default 4 threads) for async I/O
- **Node bindings** — C++ bridge between JS and system calls
- **Core modules** — `fs`, `http`, `net`, etc.

When you call `fs.readFile`, V8 hands the work off to libuv's thread pool, and your JS keeps running. When the file is ready, libuv pushes a callback to the event loop queue.

### Q2. Explain the Event Loop in detail

The event loop has **6 phases**, executed in order:

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout, setInterval callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │     pending callbacks     │  deferred I/O callbacks (e.g. TCP errors)
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
│  │       idle, prepare       │  internal use
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐      ┌───────────────┐
│  │           poll            │<─────┤   incoming:   │
│  │                           │      │  connections, │
│  └─────────────┬─────────────┘      │   data, etc.  │
│  ┌─────────────┴─────────────┐      └───────────────┘
│  │           check           │  setImmediate callbacks
│  └─────────────┬─────────────┘
│  ┌─────────────┴─────────────┐
└──┤      close callbacks      │  e.g. socket.on('close')
   └───────────────────────────┘
```

Between **each phase**, Node drains the **microtask queue**: `process.nextTick` callbacks first, then resolved Promise callbacks.

### Q3. `process.nextTick` vs `setImmediate` vs `setTimeout(fn, 0)`

```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('promise'));

console.log('sync');

// Output:
// sync
// nextTick
// promise
// timeout     (order with immediate is non-deterministic outside I/O)
// immediate
```

- `process.nextTick` — fires **before** any other I/O event, highest priority. Can starve the event loop if abused.
- **Promises (microtasks)** — drained right after `nextTick` queue.
- `setTimeout(fn, 0)` — runs in the **timers** phase; minimum delay is actually ~1ms.
- `setImmediate` — runs in the **check** phase, right after poll. Use this when you want to yield to I/O first.

**Rule of thumb:** inside an I/O callback, `setImmediate` always fires before `setTimeout(fn, 0)`.

### Q4. Is Node.js single-threaded?

**Partially.** Your JavaScript runs on a single main thread, but Node.js as a whole is multi-threaded:

- **libuv thread pool** (default 4 threads, configurable via `UV_THREADPOOL_SIZE`) handles: file I/O, DNS lookups (`dns.lookup`), CPU-heavy crypto, zlib.
- **Network I/O** uses the OS's async primitives (epoll/kqueue/IOCP) — no thread pool needed.
- **Worker threads** let you run JS on multiple threads.
- **V8 itself** uses several threads internally (GC, compilation).

### Q5. `worker_threads` vs `cluster` vs `child_process`

| Feature           | `child_process`         | `cluster`                    | `worker_threads`         |
| ----------------- | ----------------------- | ---------------------------- | ------------------------ |
| Memory            | Separate process        | Separate processes           | Shared memory possible   |
| Use case          | Spawning external bins  | Scaling HTTP servers across CPUs | CPU-heavy JS work    |
| Communication     | IPC (JSON messages)     | IPC                          | MessageChannel, SharedArrayBuffer |
| Overhead          | High (full process)     | High                         | Lower                    |

```js
// worker_threads — CPU-bound work
const { Worker, isMainThread, parentPort } = require('worker_threads');

if (isMainThread) {
  const worker = new Worker(__filename);
  worker.on('message', msg => console.log('Result:', msg));
  worker.postMessage(1_000_000);
} else {
  parentPort.on('message', (n) => {
    let sum = 0;
    for (let i = 0; i < n; i++) sum += i;
    parentPort.postMessage(sum);
  });
}
```

```js
// cluster — fork one worker per CPU for HTTP scaling
const cluster = require('cluster');
const os = require('os');

if (cluster.isPrimary) {
  os.cpus().forEach(() => cluster.fork());
  cluster.on('exit', () => cluster.fork()); // restart on crash
} else {
  require('http').createServer((req, res) => res.end('hi')).listen(3000);
}
```

### Q6. Streams — what are they and why use them?

A **stream** is an abstraction for processing data piece-by-piece instead of loading it all into memory. Everything that implements the stream interface works with `.pipe()`.

**Four types:**
- **Readable** — `fs.createReadStream`, `http.IncomingMessage`
- **Writable** — `fs.createWriteStream`, `http.ServerResponse`
- **Duplex** — both (e.g. TCP sockets)
- **Transform** — Duplex where output is derived from input (e.g. `zlib.createGzip`)

```js
// Classic streaming: file -> gzip -> file, using only a small buffer in RAM
const fs = require('fs');
const zlib = require('zlib');
const { pipeline } = require('stream/promises');

await pipeline(
  fs.createReadStream('input.txt'),
  zlib.createGzip(),
  fs.createWriteStream('input.txt.gz')
);
```

**Always prefer `pipeline` over `.pipe()`** — it handles errors and cleanup. `.pipe()` leaks on error.

### Q7. What is backpressure?

When a writable stream can't consume data as fast as it's produced, memory fills up. Streams handle this by making `.write()` return `false` — you should pause the reader until `'drain'` fires. `pipe()` and `pipeline()` do this automatically.

```js
readable.on('data', (chunk) => {
  const ok = writable.write(chunk);
  if (!ok) {
    readable.pause();
    writable.once('drain', () => readable.resume());
  }
});
```

### Q8. EventEmitter — how does it work?

`EventEmitter` is a simple pub/sub primitive. Most Node built-ins extend it (streams, HTTP, process).

```js
const { EventEmitter } = require('events');

class Order extends EventEmitter {}
const order = new Order();

order.on('created', (id) => console.log('Order', id));
order.emit('created', 42);

// Memory leak check — default max is 10 listeners per event
order.setMaxListeners(20);
```

**Gotchas:**
- Listeners run **synchronously** in registration order.
- Throwing inside a listener kills the process (unless you handle `'error'`).
- Always handle `'error'` events on emitters — unhandled `'error'` crashes Node.

### Q9. Buffer vs string, and when do you care?

A `Buffer` is a fixed-size chunk of memory outside V8's heap, useful for binary data (files, network, crypto). Strings in JS are UTF-16; converting blindly between them can corrupt binary data.

```js
const buf = Buffer.from('hello', 'utf8');
buf.toString('base64'); // "aGVsbG8="
Buffer.alloc(10);       // zero-filled (safe)
Buffer.allocUnsafe(10); // faster but may contain old memory — never send to client without overwriting
```

### Q10. CommonJS vs ES Modules

| Feature          | CommonJS              | ES Modules                 |
| ---------------- | --------------------- | -------------------------- |
| Syntax           | `require` / `module.exports` | `import` / `export` |
| Loading          | Synchronous           | Asynchronous (static)      |
| Tree-shaking     | No                    | Yes                        |
| `__dirname`      | Yes                   | No — use `import.meta.url` |
| Top-level await  | No                    | Yes                        |
| Circular deps    | Returns partial exports | Handled via live bindings |

```js
// ESM in Node — use "type": "module" in package.json, or .mjs
import { readFile } from 'node:fs/promises';
const data = await readFile('a.txt', 'utf8'); // top-level await
```

### Q11. How does `require` work?

When you call `require('x')`:
1. **Resolve** the path (core module? node_modules? relative?)
2. **Load** the file (read from disk)
3. **Wrap** it in a function: `(function (exports, require, module, __filename, __dirname) { ... })`
4. **Evaluate** the wrapped function
5. **Cache** the result in `require.cache` by absolute path

The cache means subsequent `require` calls return the **same object** — that's why modules are singletons by default.

### Q12. Memory management and garbage collection

V8 uses **generational GC**:
- **New space (young generation)** — small (1–8 MB), collected with fast **Scavenge** (copying collector). Most objects die here.
- **Old space (old generation)** — grown objects go here, collected with **Mark-Sweep-Compact** (stop-the-world pauses).

Flags you'll see in production:
- `--max-old-space-size=4096` — increase heap to 4 GB
- `--expose-gc` — enables manual `global.gc()` for profiling

### Q13. Common causes of memory leaks in Node.js

1. **Unbounded caches** (Map/object growing forever) — use `lru-cache`.
2. **Lingering event listeners** — forgetting `removeListener`, leading to `MaxListenersExceededWarning`.
3. **Global variables** accidentally holding references.
4. **Closures** capturing large objects.
5. **Timers** (`setInterval`) never cleared.
6. **Promises that never settle** — retained forever.

Debugging: take heap snapshots with `node --inspect` and Chrome DevTools, or use `clinic.js`, `heapdump`.

### Q14. What is `process` in Node.js?

A global object representing the current Node process.

```js
process.env.NODE_ENV;          // env vars
process.argv;                  // CLI args
process.exit(1);               // exit with code
process.on('uncaughtException', handler);
process.on('unhandledRejection', handler);
process.memoryUsage();         // heap info
process.cwd();                 // current working dir
process.pid;                   // process id
```

**Never `process.exit` inside a request handler** — it kills in-flight requests. Drain gracefully instead.

### Q15. Graceful shutdown — how do you implement it?

```js
const server = app.listen(3000);

const shutdown = async (signal) => {
  console.log(`Received ${signal}, shutting down...`);
  server.close(async () => {        // stop accepting new connections
    await db.close();               // close DB pool
    await redis.quit();
    process.exit(0);
  });
  // Force-kill after 10s
  setTimeout(() => process.exit(1), 10_000).unref();
};

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT',  () => shutdown('SIGINT'));
```

---

## 2. JavaScript & TypeScript Deep Dive

### Q16. Explain closures

A closure is a function that "remembers" variables from the lexical scope where it was created, even after that scope exits.

```js
function counter() {
  let count = 0;                     // captured by the returned function
  return () => ++count;
}
const c = counter();
c(); c(); c(); // 3
```

Practical use: private state, memoization, currying, event handlers with bound data.

### Q17. `this` binding in JavaScript

Rules, in order of precedence:
1. **`new` binding** — `new Foo()` → `this` is the new instance.
2. **Explicit** — `.call(obj)`, `.apply(obj)`, `.bind(obj)`.
3. **Implicit** — `obj.method()` → `this` is `obj`.
4. **Default** — plain call → `undefined` (strict) or `globalThis`.
5. **Arrow functions** don't have their own `this` — they inherit from the enclosing lexical scope.

```js
class Api {
  constructor() { this.base = '/api'; }
  // arrow method keeps `this` when passed as callback
  get = (path) => fetch(this.base + path);
}
```

### Q18. Prototypes and prototypal inheritance

Every object has a hidden `[[Prototype]]` link. Property lookup walks this chain until `null`.

```js
const animal = { eats: true };
const dog = Object.create(animal);
dog.barks = true;

dog.eats;  // true, found on prototype
Object.getPrototypeOf(dog) === animal; // true
```

ES6 `class` is syntactic sugar over prototypes.

### Q19. `var` vs `let` vs `const`

| Feature         | `var`          | `let`         | `const`         |
| --------------- | -------------- | ------------- | --------------- |
| Scope           | Function        | Block         | Block           |
| Hoisting        | Yes, initialized as `undefined` | Yes, but in temporal dead zone | Same as `let` |
| Reassignment    | Yes            | Yes           | No              |
| Global-as-property | Yes (`window.x`) | No       | No              |

### Q20. `==` vs `===`

`==` coerces types; `===` does not. Always use `===` unless you have a specific reason. One notable case: `== null` matches both `null` and `undefined`, sometimes useful.

### Q21. Explain hoisting

Variable/function declarations are "moved" to the top of their scope at compile time.
- `function` declarations: fully hoisted (callable before definition).
- `var`: hoisted and initialized as `undefined`.
- `let`/`const`/`class`: hoisted but uninitialized (Temporal Dead Zone).

### Q22. What are Symbols?

A primitive type producing unique, immutable identifiers — useful for "hidden" property keys and well-known protocol hooks (`Symbol.iterator`, `Symbol.asyncIterator`).

```js
const ID = Symbol('id');
const user = { name: 'Alice', [ID]: 123 };
Object.keys(user);           // ['name'] — Symbol is hidden
user[ID];                    // 123
```

### Q23. Generators and iterators

Generators produce a sequence lazily via `yield`.

```js
function* range(from, to) {
  for (let i = from; i <= to; i++) yield i;
}
for (const n of range(1, 5)) console.log(n);

// Async generators power async iteration
async function* paginate(url) {
  let next = url;
  while (next) {
    const res = await fetch(next).then(r => r.json());
    yield* res.items;
    next = res.nextPage;
  }
}

for await (const item of paginate('/api/users')) { ... }
```

### Q24. Debounce vs throttle

- **Debounce** — wait until user stops firing for N ms, then run once. (e.g., search-as-you-type.)
- **Throttle** — run at most once every N ms. (e.g., scroll handler.)

```js
const debounce = (fn, ms) => {
  let t;
  return (...args) => { clearTimeout(t); t = setTimeout(() => fn(...args), ms); };
};

const throttle = (fn, ms) => {
  let last = 0;
  return (...args) => {
    const now = Date.now();
    if (now - last >= ms) { last = now; fn(...args); }
  };
};
```

### Q25. `call`, `apply`, `bind`

```js
function greet(greeting, punct) { return `${greeting}, ${this.name}${punct}`; }
const user = { name: 'Alice' };
greet.call(user, 'Hi', '!');   // "Hi, Alice!"
greet.apply(user, ['Hi', '!']);// same, args as array
const bound = greet.bind(user, 'Hi');
bound('.');                     // "Hi, Alice."
```

### Q26. TypeScript — generics

```ts
function wrapInArray<T>(value: T): T[] { return [value]; }
wrapInArray(5);       // T inferred as number
wrapInArray('hi');    // T inferred as string

// Generic constraints
function byId<T extends { id: string }>(items: T[], id: string): T | undefined {
  return items.find(i => i.id === id);
}
```

### Q27. TypeScript — utility types

- `Partial<T>` — all fields optional
- `Required<T>` — all fields required
- `Pick<T, K>` / `Omit<T, K>` — select/exclude fields
- `Readonly<T>` — immutable
- `Record<K, V>` — map type
- `ReturnType<T>` / `Parameters<T>` — infer from function type
- `Awaited<T>` — unwrap a Promise

```ts
type User = { id: string; name: string; email: string };
type UserUpdate = Partial<Omit<User, 'id'>>; // id immutable, others optional
```

### Q28. `unknown` vs `any` vs `never`

- `any` — opt out of type checking (avoid).
- `unknown` — "something, but narrow before use" (safer `any`).
- `never` — impossible value; return type of functions that throw or loop forever.

---

## 3. Asynchronous Programming

### Q29. Callbacks vs Promises vs async/await

- **Callbacks** — original pattern; leads to "callback hell" and error-handling nightmares.
- **Promises** — chainable; solve composition and errors (`.catch`).
- **async/await** — syntactic sugar over Promises; looks synchronous, uses `try/catch`.

### Q30. Promise states and guarantees

A promise is in one of three states: **pending → fulfilled | rejected**. Once settled, it's immutable. `.then` callbacks always run asynchronously (as microtasks), even on an already-resolved promise.

### Q31. `Promise.all` vs `allSettled` vs `race` vs `any`

```js
await Promise.all([a, b, c]);       // reject if ANY rejects; fail-fast
await Promise.allSettled([a, b, c]);// waits for all; never rejects, returns statuses
await Promise.race([a, b]);         // first to settle (resolve OR reject)
await Promise.any([a, b]);          // first to FULFILL; rejects only if all reject
```

**Interview trap:** if one of the promises in `Promise.all` rejects, **the others keep running** — you've just lost control of them. If they have side effects (DB writes, API calls), you may want `allSettled` instead.

### Q32. How would you implement `Promise.all` yourself?

```js
function promiseAll(promises) {
  return new Promise((resolve, reject) => {
    const results = new Array(promises.length);
    let remaining = promises.length;
    if (remaining === 0) return resolve(results);
    promises.forEach((p, i) => {
      Promise.resolve(p).then(
        (val) => {
          results[i] = val;
          if (--remaining === 0) resolve(results);
        },
        reject // fail fast
      );
    });
  });
}
```

### Q33. Running async tasks with concurrency limit

```js
async function pLimit(tasks, limit) {
  const results = [];
  const executing = new Set();
  for (const task of tasks) {
    const p = Promise.resolve().then(task);
    results.push(p);
    executing.add(p);
    p.finally(() => executing.delete(p));
    if (executing.size >= limit) await Promise.race(executing);
  }
  return Promise.all(results);
}
```

Or use the `p-limit` package.

### Q34. Microtasks vs Macrotasks

- **Microtasks**: `Promise.then`, `queueMicrotask`, `process.nextTick` (nextTick has even higher priority in Node).
- **Macrotasks**: `setTimeout`, `setImmediate`, I/O callbacks.

After each macrotask, the microtask queue is **fully drained** before the next macrotask runs. That's why a misbehaving promise chain can starve I/O.

### Q35. Async error handling pitfalls

```js
// ❌ Uncaught — .then handler errors without .catch
fetchUser().then(u => JSON.parse(u.data));

// ❌ Swallowed in forEach
users.forEach(async (u) => { await save(u); }); // no one awaits these

// ✅ Proper
for (const u of users) { await save(u); }                   // sequential
await Promise.all(users.map(u => save(u)));                  // parallel
```

**Always** listen for `unhandledRejection` at the process level in production.

### Q36. Why does `async` always return a Promise?

Because ECMAScript defines `async` functions to wrap their return value in `Promise.resolve()`. Even `async function f() { return 1 }` gives you a Promise<number>.

---

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

## 5. NestJS

### Q43. What is NestJS and why use it?

A progressive Node framework written in TypeScript, inspired by Angular. It provides:
- **Dependency Injection** out of the box
- **Modular architecture** (everything is a module)
- **Decorators** for routes, validation, docs
- Built-in support for **GraphQL, WebSockets, microservices, CQRS**
- Testable by design

### Q44. Core building blocks

- **Module** — group of related code (`@Module`).
- **Controller** — HTTP route handler (`@Controller`).
- **Provider / Service** — business logic, injectable (`@Injectable`).
- **Pipe** — transform/validate input (`@UsePipes`).
- **Guard** — authorization (`@UseGuards`).
- **Interceptor** — wrap execution (logging, caching, transform response).
- **Filter** — exception handling (`@UseFilters`).

### Q45. Request lifecycle in NestJS

```
Middleware → Guards → Interceptors (before) → Pipes → Controller → Service
                           ↓
                   Interceptors (after) → Filters (if error) → Response
```

### Q46. How does DI work in NestJS?

Based on **constructor injection**. Under the hood it reads TypeScript's emitted metadata (`emitDecoratorMetadata`) to know which tokens to inject.

```ts
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User) private repo: Repository<User>,
    private logger: Logger,
  ) {}
}
```

Scopes: `DEFAULT` (singleton), `REQUEST`, `TRANSIENT`.

### Q47. Custom decorator example

```ts
export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) =>
    ctx.switchToHttp().getRequest().user,
);

@Get('me')
profile(@CurrentUser() user: User) { return user; }
```

### Q48. Guards vs Middleware vs Interceptors

| Concern                   | Use             |
| ------------------------- | --------------- |
| Generic request processing (logging, parsing) | Middleware |
| Authorization decisions ("can this run?")     | Guards     |
| Cross-cutting (timing, caching, transform)    | Interceptors |
| Transforming/validating input                 | Pipes      |
| Handling thrown exceptions                    | Filters    |

### Q49. How would you test a NestJS service?

```ts
const module = await Test.createTestingModule({
  providers: [
    UsersService,
    { provide: getRepositoryToken(User), useValue: mockRepo },
  ],
}).compile();

const service = module.get(UsersService);
```

---

## 6. REST API Design

### Q50. Design principles of REST

- **Stateless** — server holds no client state; each request is self-contained.
- **Resource-oriented URLs** — nouns, not verbs: `/users/42/orders`, not `/getUserOrders`.
- **HTTP methods have meaning** — GET, POST, PUT, PATCH, DELETE.
- **Use proper status codes**.
- **HATEOAS** — responses link to related actions (in strict REST; rarely done).
- **Content negotiation** via `Accept`.
- **Versioning** via URL, header, or media type.

### Q51. HTTP methods — semantics and idempotency

| Method | Safe | Idempotent | Typical use        |
| ------ | ---- | ---------- | ------------------ |
| GET    | Yes  | Yes        | Read               |
| HEAD   | Yes  | Yes        | Metadata only      |
| OPTIONS| Yes  | Yes        | CORS preflight     |
| POST   | No   | No         | Create, RPC-ish    |
| PUT    | No   | **Yes**    | Replace            |
| PATCH  | No   | No (usually)| Partial update    |
| DELETE | No   | Yes        | Remove             |

**Idempotent** = calling it N times has the same effect as calling it once.
**Safe** = no server state change.

### Q52. Status codes that matter

- `200` OK — generic success
- `201` Created — new resource; include `Location` header
- `202` Accepted — async, processing later
- `204` No Content — success, empty body (DELETEs)
- `301/302/307/308` — redirects (308 keeps method and body)
- `400` Bad Request — malformed input
- `401` Unauthorized — not authenticated
- `403` Forbidden — authenticated but not allowed
- `404` Not Found
- `409` Conflict — state collision (e.g., duplicate email)
- `410` Gone
- `422` Unprocessable Entity — semantic validation failure
- `429` Too Many Requests — rate-limited
- `500` Internal Server Error
- `502/503/504` — upstream/availability problems

### Q53. API versioning strategies

1. **URI path**: `/v1/users` — most visible, easiest.
2. **Header**: `Accept: application/vnd.app.v2+json`.
3. **Query param**: `?version=2` — discouraged.

### Q54. Pagination strategies

- **Offset-based** — `?limit=20&offset=100`. Simple but breaks with inserts and scales poorly.
- **Cursor-based (keyset)** — `?limit=20&after=<encoded_cursor>`. Stable, fast for big datasets. **Prefer this at scale.**

```
GET /events?limit=20
{ "data": [...], "next": "eyJpZCI6IjU2In0=" }
```

### Q55. Idempotency for POST

POST isn't idempotent, but clients may retry on network failure. Support an `Idempotency-Key` header: on the server, store the key→response mapping for 24h so retries return the original result instead of creating duplicates. (Stripe's standard.)

### Q56. How would you design `POST /transfers`?

- Validate `amount > 0`, distinct accounts, currencies match.
- Require `Idempotency-Key`.
- Do it in a DB transaction: lock both accounts (ordered by ID to avoid deadlocks), check balance, update, insert ledger entries.
- Return `201` with transfer ID; enqueue post-processing (notifications) to a queue, never inline.

---

## 7. Authentication, Authorization & Security

### Q57. JWT — structure and how it works

A JWT has three base64url-encoded parts separated by dots: `header.payload.signature`.

```
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkFsaWNlIn0.d8Zj...
```

- **Header** — `{ "alg": "HS256", "typ": "JWT" }`
- **Payload** — claims: `sub`, `iat`, `exp`, `iss`, `aud`, plus custom
- **Signature** — HMAC or RSA over `base64(header).base64(payload)`

The server **verifies** the signature with a secret (HS256) or public key (RS256). If valid and not expired, trust the payload.

### Q58. JWT pros, cons, and pitfalls

**Pros:** stateless, easy to use across services, self-contained.
**Cons:**
- Cannot be revoked before expiry without extra infra (blocklist).
- If the payload is large, it bloats every request.
- Secret leak = total compromise.
- Algorithm confusion attack (`alg: none`, swapping RS256→HS256 with public key as secret) — always pin the algorithm on verify.

### Q59. Access tokens vs refresh tokens

- **Access token** — short-lived (5–15 min), sent on every request.
- **Refresh token** — long-lived (days/weeks), stored securely, used to mint a new access token.

Typical flow:
1. Login → server returns `{ accessToken, refreshToken }`.
2. On 401, client uses refresh token at `/auth/refresh`.
3. Rotate refresh token on each use; store rotations to detect theft.

**Where to store tokens in a browser?**
- Access token → memory (never localStorage; XSS steals it).
- Refresh token → **httpOnly, Secure, SameSite=Strict cookie**.

### Q60. Session-based auth vs JWT — when to use which?

| Factor        | Sessions                          | JWT                            |
| ------------- | --------------------------------- | ------------------------------ |
| Storage       | Server (DB/Redis)                 | Client                         |
| Revocation    | Trivial (delete session)          | Hard without blocklist         |
| Scalability   | Needs sticky session or shared store | Stateless                  |
| Use case      | Web app with a single backend     | Microservices, mobile, 3rd-party APIs |

For a classic web app, cookies + server session is usually the simpler, safer choice.

### Q61. OAuth 2.0 — core flows

- **Authorization Code + PKCE** — for web/SPAs/mobile. The gold standard.
- **Client Credentials** — machine-to-machine.
- **Device Code** — smart TVs, CLI tools.
- **Refresh Token** — getting new access tokens.
- **Implicit** — deprecated; don't use.
- **Password** — deprecated; don't use.

### Q62. Password hashing

- **Never** store plaintext. **Never** use MD5 or SHA-1 or SHA-256 directly.
- Use **bcrypt** (good default), **scrypt**, or **argon2** (best).

```js
import bcrypt from 'bcrypt';
const hash = await bcrypt.hash(password, 12);  // 12 rounds = ~250ms
const ok = await bcrypt.compare(password, hash);
```

Bcrypt generates the salt; the hash encodes algorithm, rounds, salt, and digest in one string.

### Q63. OWASP Top 10 — what does a senior need to know?

1. **Broken Access Control** — always check authz **per resource**, not just "is logged in".
2. **Cryptographic Failures** — HTTPS everywhere, don't roll your own crypto.
3. **Injection** — parameterized queries, never string-concat SQL.
4. **Insecure Design**.
5. **Security Misconfiguration** — disable stack traces in prod, lock down defaults.
6. **Vulnerable/Outdated Components** — `npm audit`, Dependabot.
7. **Identification and Authentication Failures**.
8. **Software and Data Integrity Failures** — npm supply chain, SRI.
9. **Security Logging and Monitoring Failures**.
10. **Server-Side Request Forgery (SSRF)** — validate outgoing URLs, block internal IPs.

### Q64. SQL injection prevention

**Always use parameterized queries.**

```js
// ❌ Vulnerable
db.query(`SELECT * FROM users WHERE email = '${email}'`);

// ✅ Parameterized
db.query('SELECT * FROM users WHERE email = $1', [email]);
```

ORMs do this automatically, but the moment someone uses a raw query, you're back at risk.

### Q65. XSS and CSRF — what are they?

**XSS (Cross-Site Scripting)** — attacker injects JS into a page another user views. Defense: escape output, use Content-Security-Policy, `httpOnly` cookies.

**CSRF (Cross-Site Request Forgery)** — attacker tricks an authenticated user's browser into submitting a request. Defense: `SameSite` cookies, CSRF tokens, check `Origin`/`Referer`.

### Q66. CORS explained

CORS is a browser mechanism to allow a site on origin A to call an API on origin B. For "simple" requests, the browser just checks the `Access-Control-Allow-Origin` response header. For "preflighted" requests (custom headers, non-GET/POST, etc.), the browser first sends an `OPTIONS` request.

```js
app.use(cors({
  origin: 'https://app.example.com',
  credentials: true,              // allow cookies
  methods: ['GET','POST','PUT','DELETE'],
}));
```

Common mistake: `Access-Control-Allow-Origin: *` **with** `credentials: true` — browsers reject this; you must echo the specific origin.

### Q67. Rate limiting strategies

- **Fixed window** — N requests per minute. Bursts at boundaries.
- **Sliding window log** — accurate, memory-hungry.
- **Sliding window counter** — balance of both.
- **Token bucket** — allows bursts up to bucket size.
- **Leaky bucket** — enforces constant rate.

In Node: `express-rate-limit` + Redis store for distributed deployments.

### Q68. Common Node-specific vulnerabilities

- **Prototype pollution** — merging untrusted objects into `Object.prototype`.
- **ReDoS** — catastrophic backtracking in regex against attacker input.
- **Command injection** via `child_process.exec` with interpolated user input — use `execFile` + args array.
- **Path traversal** — accepting `../../etc/passwd` into `fs.readFile` — always `path.resolve` and verify prefix.
- **Open redirect** — `res.redirect(req.query.next)` → validate allowlist.

---

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

## 9. ORMs — Prisma, Sequelize, TypeORM

### Q81. Compare the three

| Feature              | Prisma                   | Sequelize               | TypeORM                |
| -------------------- | ------------------------ | ----------------------- | ---------------------- |
| Type safety          | Excellent (generated)    | Weak                    | Good (decorators)      |
| Schema source        | `schema.prisma` DSL      | JS models               | Decorated classes      |
| Migrations           | First-class              | CLI                     | CLI                    |
| Active Record / DM   | Data Mapper              | Active Record           | Both                   |
| Raw SQL escape       | `$queryRaw`              | `sequelize.query`       | `query`                |
| Community            | Fastest growing          | Oldest, huge            | Stable, NestJS default |
| Downsides            | Extra generate step, opinionated | Type-unsafe, clunky | Maintenance concerns |

### Q82. Prisma example

```prisma
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  posts     Post[]
  createdAt DateTime @default(now())
}
```

```ts
const user = await prisma.user.create({
  data: { email: 'a@b.com', posts: { create: [{ title: 'Hi' }] } },
  include: { posts: true },
});
```

### Q83. Migrations — best practices

- **Never** edit a migration that's been applied in production. Create a new one.
- Forward-compatible deploys: **expand → migrate → contract**.
  1. Add new column nullable.
  2. Dual-write.
  3. Backfill.
  4. Switch reads.
  5. Drop old column in a later deploy.
- Keep migrations small and fast — avoid long table locks.

### Q84. Transactions with an ORM

Prisma:
```ts
await prisma.$transaction(async (tx) => {
  await tx.account.update({ where: { id: from }, data: { balance: { decrement: amount } } });
  await tx.account.update({ where: { id: to   }, data: { balance: { increment: amount } } });
  if ((await tx.account.findUnique({ where: { id: from } }))!.balance < 0) throw new Error('overdraft');
});
```

TypeORM has `queryRunner.startTransaction()` / `commitTransaction()` and `dataSource.transaction(cb)`.

### Q85. When would you drop the ORM and write raw SQL?

- Reports with complex joins, window functions, CTEs.
- Bulk operations (`INSERT ... ON CONFLICT`, `COPY`).
- Queries where the ORM generates inefficient SQL (check `EXPLAIN`).
- DB-specific features (PostgreSQL ranges, JSONB paths, `LATERAL` joins).

---

## 10. GraphQL

### Q86. What is GraphQL and how is it different from REST?

A query language and runtime where clients specify **exactly** which fields they need. One endpoint, strongly-typed schema.

| Aspect        | REST                       | GraphQL                             |
| ------------- | -------------------------- | ----------------------------------- |
| Endpoints     | Many                       | One                                 |
| Over/underfetching | Common                | Minimal                             |
| Versioning    | Explicit (`/v1/`)          | Evolve schema (deprecate fields)    |
| Caching       | HTTP caching out-of-box    | Harder; needs extra infra           |
| Error codes   | HTTP statuses              | Always 200; errors in body          |
| File uploads  | Trivial                    | Requires extension (`graphql-upload`) |

### Q87. Schema, queries, mutations, subscriptions

```graphql
type User  { id: ID!, email: String!, posts: [Post!]! }
type Post  { id: ID!, title: String!, author: User! }

type Query { user(id: ID!): User }
type Mutation { createPost(input: CreatePostInput!): Post! }
type Subscription { postAdded: Post! }
```

### Q88. Resolvers and the N+1 problem

Each field can have a resolver. If a query returns 100 posts, the naive `post.author` resolver runs 100 SQL queries — the GraphQL N+1 problem.

**Fix: DataLoader.** It batches and caches (per-request).

```ts
const userLoader = new DataLoader(async (ids) => {
  const users = await db.user.findMany({ where: { id: { in: ids as string[] } } });
  const map = new Map(users.map(u => [u.id, u]));
  return ids.map(id => map.get(id as string));
});

// resolver
author: (post, _, ctx) => ctx.loaders.user.load(post.authorId),
```

### Q89. Apollo Server setup (Node)

```ts
const server = new ApolloServer({ typeDefs, resolvers });
const { url } = await startStandaloneServer(server, {
  context: async ({ req }) => ({
    user: await getUserFromToken(req.headers.authorization),
    loaders: makeLoaders(),
  }),
});
```

### Q90. Authorization in GraphQL

- **Field-level** — check in resolvers (or directives like `@auth(role: ADMIN)`).
- **Schema-level** — gate entire types.
- Keep **business rules** in services, not resolvers.
- Avoid exposing internal IDs/raw DB shapes — use DTOs.

### Q91. Schema stitching vs Federation

- **Stitching** — merge remote schemas into a gateway. Older, simpler, less scalable.
- **Federation (Apollo)** — each subgraph owns its types; gateway composes them. Better for microservices.

### Q92. GraphQL downsides / gotchas

- Query complexity → potential DoS. Use depth limiting, cost analysis, persisted queries.
- HTTP caching is hard (queries are POSTs with varied bodies).
- Client must know GraphQL; learning curve.
- Introspection can leak schema — disable in prod if unused.

---

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

## 12. System Design & Architecture

### Q100. Monolith vs Microservices

**Monolith pros:** simpler ops, simpler transactions, lower latency between modules, easier local dev.
**Microservices pros:** independent deploys, scale critical services separately, team autonomy, fault isolation.
**Cost:** network complexity, data consistency, observability, deployment infrastructure.

Rule: start with a **modular monolith**. Split services when team/scale friction demands it.

### Q101. Synchronous vs asynchronous communication

- **Sync (HTTP, gRPC)** — simple, but couples availability.
- **Async (queue, event bus)** — decoupled, retriable, but eventual consistency.

Rule: **orchestration for synchronous workflows, choreography + events for decoupled systems.**

### Q102. Message queues — why use them?

- Decouple producers and consumers.
- Handle traffic spikes (buffering).
- Retries with backoff and DLQ (dead-letter queue).
- Enable async workflows (email, thumbnails, reports).

```
Web App → SQS → Worker → DB + S3
```

### Q103. At-least-once vs exactly-once delivery

Most queues give **at-least-once**. Design your consumers to be **idempotent** — safe to process the same message twice. Use a unique message ID + dedupe store (Redis set, DB constraint).

### Q104. Caching strategies

- **Cache-aside (lazy loading)** — app checks cache, then DB on miss, then writes back.
- **Read-through** — cache sits in front of DB transparently.
- **Write-through** — write to cache and DB simultaneously.
- **Write-behind** — write to cache, flush to DB async.

Pitfalls: stale data, cache stampede, thundering herd. Mitigations: short TTLs, jittered TTLs, request coalescing (single-flight).

### Q105. Redis — use cases

- Session store
- Rate limiter (atomic `INCR` + `EXPIRE`)
- Distributed lock (`SET NX PX`; for correctness, use Redlock)
- Pub/sub
- Queues (`LPUSH`/`BRPOP`, or Streams)
- Leaderboards (sorted sets)
- Geospatial queries
- Caching

### Q106. Horizontal vs vertical scaling

- **Vertical** — bigger box. Cheap until it isn't; hard limit.
- **Horizontal** — more boxes. Requires stateless design, load balancing, shared data layer.

### Q107. Load balancer layers

- **L4 (TCP)** — fast, port-based (AWS NLB).
- **L7 (HTTP)** — can route by path/header/cookie, TLS termination, WAF (AWS ALB, NGINX).

Session stickiness only when needed (WebSockets); otherwise prefer stateless + external session store.

### Q108. Database scaling

- **Vertical** — bigger instance.
- **Read replicas** — route reads to replicas, writes to primary.
- **Partitioning** — split tables by range/hash (same DB).
- **Sharding** — split across DBs.
- **CQRS** — separate read model (denormalized, indexed for queries) from write model.

### Q109. Eventual consistency — how do you deal with it?

- Show optimistic UI updates and reconcile.
- Use **event sourcing** to rebuild views.
- Expose **idempotent operations** and retries.
- Use **sagas** or the **outbox pattern** for distributed transactions.

### Q110. Outbox pattern

When you need to both write to the DB and publish an event without losing either:

1. In the same DB transaction, insert the event into an `outbox` table.
2. A separate poller reads `outbox`, publishes to the broker, marks as sent.

This avoids the dual-write problem (DB commits but broker publish fails, or vice versa).

### Q111. Circuit breaker pattern

Wrap remote calls with a breaker that trips (fails fast) after repeated failures. States: **closed → open → half-open**. Prevents cascading failures and gives the downstream service time to recover. Libraries: `opossum`.

### Q112. Design Twitter (high level)

- **Write path**: tweet → append to user's timeline → fan-out on write to a limited follower set. For celebrities (> N followers) use fan-out on read.
- **Timeline**: precomputed in Redis; merged with recent live tweets.
- **Storage**: Cassandra/Dynamo for tweets, sharded by user ID.
- **Search**: Elasticsearch.
- **Media**: S3 + CDN.
- **Rate limits**: Redis-backed token bucket per user.

### Q113. Design a URL shortener

- Generate short keys with base62 over an auto-increment ID, or use a hash (collision handling needed).
- Store in key-value DB (Redis/Dynamo) for O(1) lookup.
- Edge-cache popular URLs in CDN.
- Analytics via async queue → data warehouse.
- Expirations via TTL.

---

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

## 14. Testing

### Q119. Testing pyramid

- **Unit** (70%) — pure functions, small, fast, isolated.
- **Integration** (20%) — modules together (service + DB, controller + route).
- **E2E** (10%) — full system from an HTTP client.

Inversion (ice-cream cone) — too many slow E2E tests, too few units — is a common smell.

### Q120. Jest basics + mocking

```ts
import { sum } from './sum';

jest.mock('./db');
const db = require('./db');

test('sums', () => {
  db.load.mockResolvedValue([1, 2, 3]);
  expect(sum([1,2])).toBe(3);
});
```

Prefer **dependency injection** over deep mocking — it keeps production code simple and tests honest.

### Q121. Integration testing a REST API

```ts
import request from 'supertest';
import { createApp } from '../src/app';

it('POST /users creates a user', async () => {
  const res = await request(app).post('/users').send({ email: 'a@b.c' });
  expect(res.status).toBe(201);
  expect(res.body).toMatchObject({ email: 'a@b.c' });
});
```

Run DB tests against a real Postgres in Docker (`testcontainers-node`) — better than mocking the DB layer.

### Q122. TDD — when is it worth it?

- When requirements are clear and well-defined.
- When the component is pure/algorithmic.
- When refactoring legacy code (tests protect behavior).

Skip TDD for throwaway scripts, UI experiments, or when you're still exploring the design.

### Q123. Flaky tests — common causes and fixes

- Time-dependence — mock `Date`, use fake timers.
- Network calls — mock HTTP (`nock`, `msw`).
- Order-dependence — clean state between tests.
- Race conditions in async — await everything, use deterministic scheduling.

---

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

## 16. Coding Challenges

### Q130. Implement `once(fn)` — run a function only once

```js
function once(fn) {
  let called = false, result;
  return (...args) => {
    if (!called) { called = true; result = fn(...args); }
    return result;
  };
}
```

### Q131. Implement `memoize(fn)`

```js
function memoize(fn) {
  const cache = new Map();
  return (...args) => {
    const key = JSON.stringify(args);
    if (!cache.has(key)) cache.set(key, fn(...args));
    return cache.get(key);
  };
}
```

### Q132. Implement a simple EventEmitter

```js
class MyEmitter {
  #listeners = new Map();
  on(event, fn)   { (this.#listeners.get(event) ?? this.#listeners.set(event, []).get(event)).push(fn); return this; }
  off(event, fn)  { const ls = this.#listeners.get(event); if (ls) this.#listeners.set(event, ls.filter(l => l !== fn)); }
  emit(event, ...args) { (this.#listeners.get(event) ?? []).slice().forEach(fn => fn(...args)); }
  once(event, fn) { const wrap = (...a) => { this.off(event, wrap); fn(...a); }; this.on(event, wrap); }
}
```

### Q133. Retry with exponential backoff

```js
async function retry(fn, { attempts = 5, base = 100, factor = 2, jitter = true } = {}) {
  let attempt = 0;
  while (true) {
    try { return await fn(); }
    catch (e) {
      if (++attempt >= attempts) throw e;
      const delay = base * factor ** (attempt - 1) * (jitter ? (0.5 + Math.random()) : 1);
      await new Promise(r => setTimeout(r, delay));
    }
  }
}
```

### Q134. Deep clone

```js
// Modern, handles cycles, Maps, Sets, Dates, ArrayBuffers
const copy = structuredClone(obj);
```

Interview-style manual version:
```js
function deepClone(o, seen = new WeakMap()) {
  if (o === null || typeof o !== 'object') return o;
  if (seen.has(o)) return seen.get(o);
  const out = Array.isArray(o) ? [] : {};
  seen.set(o, out);
  for (const k of Reflect.ownKeys(o)) out[k] = deepClone(o[k], seen);
  return out;
}
```

### Q135. Flatten a nested array

```js
// Built-in
arr.flat(Infinity);

// Manual
const flatten = (a) => a.reduce((acc, v) => acc.concat(Array.isArray(v) ? flatten(v) : v), []);
```

### Q136. Implement `pipe(...fns)` and `compose(...fns)`

```js
const pipe    = (...fns) => (x) => fns.reduce((acc, fn) => fn(acc), x);
const compose = (...fns) => (x) => fns.reduceRight((acc, fn) => fn(acc), x);

const toUpperTrim = pipe((s) => s.trim(), (s) => s.toUpperCase());
toUpperTrim('  hi  '); // "HI"
```

### Q137. Fix an async `forEach` that doesn't wait

```js
// ❌ runs in parallel, nothing awaited
arr.forEach(async (x) => await doWork(x));

// ✅ sequential
for (const x of arr) await doWork(x);

// ✅ parallel
await Promise.all(arr.map(doWork));

// ✅ parallel with concurrency limit
import pLimit from 'p-limit';
const limit = pLimit(5);
await Promise.all(arr.map(x => limit(() => doWork(x))));
```

### Q138. Throttle concurrent API calls while preserving order of results

```js
async function mapConcurrent(items, fn, concurrency = 5) {
  const results = new Array(items.length);
  let i = 0;
  const workers = Array.from({ length: concurrency }, async () => {
    while (i < items.length) {
      const idx = i++;
      results[idx] = await fn(items[idx], idx);
    }
  });
  await Promise.all(workers);
  return results;
}
```

### Q139. Build a simple rate limiter (token bucket)

```js
class TokenBucket {
  constructor(capacity, refillPerSec) {
    this.capacity = capacity;
    this.tokens = capacity;
    this.refillPerSec = refillPerSec;
    this.last = Date.now();
  }
  take() {
    const now = Date.now();
    this.tokens = Math.min(this.capacity, this.tokens + ((now - this.last) / 1000) * this.refillPerSec);
    this.last = now;
    if (this.tokens >= 1) { this.tokens -= 1; return true; }
    return false;
  }
}
```

### Q140. LRU Cache

```js
class LRU {
  constructor(max) { this.max = max; this.map = new Map(); }
  get(k) {
    if (!this.map.has(k)) return undefined;
    const v = this.map.get(k);
    this.map.delete(k); this.map.set(k, v); // move to tail
    return v;
  }
  set(k, v) {
    if (this.map.has(k)) this.map.delete(k);
    else if (this.map.size >= this.max) this.map.delete(this.map.keys().next().value);
    this.map.set(k, v);
  }
}
```

### Q141. Big-O — know the basics

- Array push/pop: O(1). unshift/shift: O(n).
- `Map`/`Set`: O(1) avg for get/set/delete.
- Object property access: O(1) avg.
- Sort: O(n log n).
- Searching sorted array with binary search: O(log n).
- Graph BFS/DFS: O(V + E).

---

## 17. Behavioral / Senior-Level Questions

### Q142. "Tell me about a time you had a serious production incident."

Use **STAR** (Situation, Task, Action, Result). A good story includes:
- What signal tipped you off (alerts, user reports).
- How you triaged (rollback vs. forward-fix).
- What root cause analysis revealed.
- Postmortem actions (blameless, systemic fixes, not "be more careful").

### Q143. "How do you mentor juniors?"

Things to mention:
- Pair programming.
- PR reviews that explain *why*, not just *what*.
- Giving them safe, well-scoped tasks before ambitious ones.
- Encouraging them to propose solutions, then guiding.
- Dev2dev talks, knowledge sharing.

### Q144. "How do you make technical decisions in a team?"

- Write an **ADR** (Architecture Decision Record) or design doc.
- Present options with tradeoffs (latency, cost, complexity).
- Engage stakeholders early; disagree-and-commit when needed.
- Measure outcomes; revisit if data says otherwise.

### Q145. "How do you balance speed vs quality?"

- Tests for the **critical path**; less coverage in experimental areas.
- Feature flags for risky changes.
- Fast rollback paths.
- Tech-debt budget in sprints, not "someday".

### Q146. "A colleague insists on a bad technical decision. What do you do?"

- Ask questions first — make sure I understand their reasoning.
- Present concrete counter-evidence (benchmarks, postmortems, industry practice).
- If still deadlocked, escalate the decision owner (tech lead, architect).
- Once decided, support the decision even if I disagreed — but log the risk.

### Q147. "Why are you leaving your current job?"

Be honest but positive. Don't bash your employer. Frame in terms of what you're **looking for** (larger scale, more ownership, new domain).

### Q148. Questions to ask the interviewer

- How is the engineering team organized, and what does decision-making look like?
- What does the on-call rotation look like?
- How do you balance product work vs. tech debt?
- What's the biggest technical challenge the team is facing right now?
- What does success look like in this role after 3 / 6 / 12 months?
- How is feedback given (1:1s, review cycles)?
- What's the deployment frequency, and who owns production?

---

## Final Tips

1. **When you don't know something, say so** — then reason out loud. Seniors aren't expected to know everything; they're expected to think clearly and know their limits.
2. **Think in tradeoffs, not absolutes.** "It depends, because..." is a senior answer. "X is always better" is not.
3. **Mention failure modes.** When you propose a solution, volunteer how it could break and how you'd detect/mitigate it.
4. **Use real numbers.** "p95 latency went from 450ms to 80ms after we added the Redis cache" lands far better than "it got faster".
5. **Drive the conversation.** Ask clarifying questions at the start of system design problems — requirements, scale, SLAs — before diving in.
6. **Practice out loud.** Explaining concepts verbally is a different skill from understanding them.
7. **Prepare 3–5 stories** you can adapt to behavioral questions (incident, conflict, leadership, tradeoff, learning).

Good luck — you've got this.
