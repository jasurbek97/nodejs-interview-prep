# Node.js In Depth

### What the runtime actually does when you run `node server.js`.

---

## Preface

The JavaScript book taught you the language. This one teaches you the runtime that runs the language on servers: what the process does, how I/O becomes events, why streams exist, when to reach for Worker Threads instead of Cluster, and what the phrase "non-blocking I/O" really means when you're staring at a slow endpoint at 2 a.m.

Same structure as before: **What it is**, **Why you care**, **How it works**, **Watch out**.

---

# Part I — The Runtime

## Chapter 1. What Node.js Really Is

### What it is

Node.js is a program that embeds **V8** (Google's JavaScript engine) and wires it up to **libuv** (a C library for async I/O and a thread pool) plus a set of C++ bindings exposing OS primitives (files, sockets, timers, crypto, DNS, etc.). Your JS is the script language; V8 runs it; libuv does the actual work that JS can't do itself.

### Why you care

"Single-threaded, non-blocking, event-driven" is three phrases that mean the same thing: *your JavaScript runs on one thread, and everything slow (disk, network, DNS, crypto) is delegated elsewhere*. Once you see the split, Node stops being magic.

### The components

```
┌───────────────────────────────────────────────────┐
│  Your JS code                                     │
├───────────────────────────────────────────────────┤
│  Node.js core modules (fs, http, net, crypto…)    │
├──────────────┬────────────────────────────────────┤
│              │   C++ bindings                     │
│     V8       ├────────────────────────────────────┤
│  (JS engine) │   libuv (event loop, thread pool,  │
│              │   async I/O, timers)               │
├──────────────┴────────────────────────────────────┤
│  Operating system (epoll / kqueue / IOCP)         │
└───────────────────────────────────────────────────┘
```

- **V8** compiles and runs JS.
- **libuv** implements the event loop, a thread pool (default 4 threads), and cross-platform async I/O.
- **C++ bindings** glue them: `fs.readFile` is a JS function that calls a C++ function that asks libuv to read the file.

### Watch out

- "Non-blocking" applies to I/O, not CPU. A tight `while` loop in JS blocks the event loop. Crypto, image processing, JSON.parse on huge strings — all can block.
- Node is the *runtime*; npm is the *package manager*. They're separate projects that ship together.

---

## Chapter 2. Node vs the Browser

### What it is

Both run JS on V8, but they expose different globals and APIs, and they run in different trust and security models.

### The table

| | Browser | Node |
|---|---|---|
| Global object | `window` (also `globalThis`) | `global` (also `globalThis`) |
| Module system | ESM by default | CJS by default; ESM opt-in |
| DOM / `fetch` | Yes / Yes | No DOM; `fetch` since Node 18 |
| File system | Via `File` / `FileReader` | `fs`, `fs/promises` |
| Timers | `setTimeout` returns a number | Returns a `Timeout` object |
| Sandboxed | Yes — origin-isolated | No — full OS access |
| Entry point | `<script>` | `node file.js` |
| Event loop | Integrated with rendering | libuv, no rendering |

### Watch out

- `fetch`, `URL`, `FormData`, `Blob`, `crypto.subtle` are all available in modern Node — but some edge behaviors differ.
- `Buffer` is Node-specific. In browsers use `Uint8Array` or `Blob`.
- In Node 18+ you can mostly write universal code, but don't assume — test both sides when it matters.

---

## Chapter 3. The Event Loop (libuv)

### What it is

A phase machine that runs one iteration ("tick") at a time. Each phase has its own queue; callbacks added during a phase run in that or the next tick depending on the phase.

### The six phases

```
   ┌───────────────────────────┐
┌─>│           timers          │  setTimeout, setInterval callbacks whose time is up
│  ├───────────────────────────┤
│  │     pending callbacks     │  deferred I/O (rare: e.g., TCP errors)
│  ├───────────────────────────┤
│  │       idle, prepare       │  internal
│  ├───────────────────────────┤
│  │                           │  ← wait here for I/O (epoll/kqueue)
│  │           poll            │  run I/O callbacks
│  │                           │  
│  ├───────────────────────────┤
│  │           check           │  setImmediate callbacks
│  ├───────────────────────────┤
└──┤      close callbacks      │  'close' events (sockets, handles)
   └───────────────────────────┘
```

**Between every phase**, Node drains:
1. `process.nextTick` queue (highest priority).
2. Microtask queue (Promise reactions, `queueMicrotask`).

Then the loop moves on.

### Example: the classic `setTimeout` vs `setImmediate` race

```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
// Order is NOT guaranteed outside an I/O callback.
// Inside an I/O callback, setImmediate always wins:
fs.readFile('x', () => {
  setTimeout(() => console.log('t'), 0);
  setImmediate(() => console.log('i'));
  // 'i' first — poll phase enters check before timers on next tick
});
```

### Use case: knowing what blocks

```js
const t0 = Date.now();
fs.readFile('huge.log', () => {
  console.log('I/O done at', Date.now() - t0);
});
// Tight loop blocks the event loop — I/O callback waits
while (Date.now() - t0 < 2000) { /* blocks */ }
```

### Watch out

- `process.nextTick` runs **before** Promise microtasks. Recursive `nextTick` can starve I/O.
- The thread pool (default 4) handles fs, crypto, DNS, zlib. CPU-bound crypto can saturate it — bump `UV_THREADPOOL_SIZE` or move work to Worker Threads.

---

## Chapter 4. Timers in Detail

### What it is

`setTimeout`, `setInterval`, `setImmediate`, `process.nextTick`, `queueMicrotask` — five ways to defer work, with different phases and priorities.

### Priority order

```
currently running stack
  ├─ process.nextTick queue     ← drained first, fully
  ├─ microtasks (Promise)       ← drained next, fully
  ├─ timers (setTimeout)        ← phase: timers
  ├─ I/O callbacks              ← phase: poll
  ├─ setImmediate               ← phase: check
  └─ close events               ← phase: close
```

### Example

```js
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
console.log('sync');

// sync
// nextTick
// promise
// timeout   (or immediate first; order non-deterministic here)
// immediate
```

### Use case: deferring a heavy computation so a response can flush

```js
function defer(fn) { setImmediate(fn); }

app.post('/process', (req, res) => {
  res.json({ queued: true });
  defer(() => heavyJob(req.body));          // returns to client immediately
});
```

### Watch out

- `setTimeout(fn, 0)` is clamped to ~1ms minimum. Use `setImmediate` or `queueMicrotask` for "next tick".
- `setInterval` drifts — use `setTimeout` that reschedules if precision matters.

---

# Part II — Modules and Process

## Chapter 5. CommonJS and ESM in Node

### What it is

Node speaks both module systems. Which one your file uses is decided by extension and `package.json` settings.

### The rules

| File | Module system |
|---|---|
| `.mjs` | ESM |
| `.cjs` | CJS |
| `.js` with `"type": "module"` in nearest `package.json` | ESM |
| `.js` otherwise | CJS |

### Examples

```js
// CJS
const fs = require('node:fs');           // sync
module.exports = { add: (a, b) => a + b };
```

```js
// ESM
import fs from 'node:fs';                // async under the hood
export function add(a, b) { return a + b; }
export default add;
```

### Interop

- ESM can `import` CJS: you get `module.exports` as the default, and named exports when statically analyzable.
- CJS loading ESM was impossible for years; modern Node (22+) supports synchronous ESM via `require()` with flags; prefer dynamic `import()` for compatibility.

### Use case: `import.meta.url` for the current file path

```js
import { fileURLToPath } from 'node:url';
const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);
```

(`__filename`/`__dirname` exist automatically in CJS; ESM needs the above.)

### Watch out

- `require.cache` lets you inspect/evict module caches (handy in tests). ESM has no equivalent — modules load once per process.
- Top-level `await` works in ESM only; the whole module becomes async.
- Named imports from CJS sometimes fail if the library does `module.exports = something` (not `module.exports.x = ...`). Use `import pkg from 'x'; const { a } = pkg;`.

---

## Chapter 6. The `process` Object

### What it is

A global object representing the current Node process. Environment, CLI args, stdio, exit, signals, resource usage — all live here.

### Why you care

Every production concern — reading config, handling SIGTERM, exiting cleanly, measuring memory — goes through `process`.

### The essentials

```js
process.argv;                           // ['node', '/path/script.js', ...args]
process.env.NODE_ENV;                   // string or undefined
process.cwd();                          // current working directory
process.pid;                            // process id
process.platform;                       // 'darwin' | 'linux' | 'win32'
process.version;                        // 'v22.2.0'
process.memoryUsage();                  // { rss, heapTotal, heapUsed, external }
process.uptime();                       // seconds since start

process.stdout.write('no newline');
process.stderr.write('error');
process.stdin.on('data', (chunk) => { /* ... */ });
```

### Signals and exit

```js
process.on('SIGTERM', shutdown);
process.on('SIGINT', shutdown);         // Ctrl-C
process.on('uncaughtException', (err) => { log(err); process.exit(1); });
process.on('unhandledRejection', (reason) => { log(reason); process.exit(1); });

async function shutdown(sig) {
  log(`got ${sig}`);
  await server.close();
  await db.destroy();
  process.exit(0);
}
```

### Watch out

- `process.exit()` terminates **synchronously** — pending I/O is dropped. Prefer closing resources and letting the loop drain naturally.
- Environment variables are strings. Always parse: `process.env.PORT ? Number(process.env.PORT) : 3000`.

---

## Chapter 7. EventEmitter

### What it is

A base class for objects that emit named events. Most Node core objects (streams, HTTP servers, processes) inherit from it.

### How it works

```js
import { EventEmitter } from 'node:events';

class Counter extends EventEmitter {
  inc() { this.n = (this.n || 0) + 1; this.emit('tick', this.n); }
}

const c = new Counter();
c.on('tick', (n) => console.log('n =', n));
c.inc(); c.inc();
// n = 1
// n = 2
```

### The methods you'll use

- `.on(event, fn)` / `.off(event, fn)` — listen / remove.
- `.once(event, fn)` — auto-remove after first fire.
- `.emit(event, ...args)` — synchronous call to all listeners.
- `.removeAllListeners(event?)`.
- `.setMaxListeners(n)` — default warn at 10 (indicates a possible leak).

### Use case: domain events in a service

```js
class OrderService extends EventEmitter {
  async create(data) {
    const order = await db.orders.insert(data);
    this.emit('order.created', order);
    return order;
  }
}
```

Decouples side effects: analytics, webhooks, email, etc., subscribe independently.

### Watch out

- **Emit is synchronous.** Throwing inside a listener can crash the emitter. Wrap listener bodies in try/catch or use `once('error', ...)`.
- Listener leaks: always remove listeners when the subject outlives them, or the MaxListenersExceeded warning will appear.
- For async event handlers, consider `events.on(emitter, name)` — an async iterator of events.

---

# Part III — I/O and Streams

## Chapter 8. Buffer

### What it is

A fixed-size chunk of raw bytes, allocated **outside** V8's heap. The pre-`Uint8Array` way Node handled binary data — kept for compatibility and zero-copy interop with libuv.

### Why you care

Everything that hits the wire or the disk starts as a Buffer. Knowing how to slice, encode, and concatenate them without unnecessary copies matters for throughput.

### How it works

```js
const b = Buffer.from('hello', 'utf8');  // 5 bytes
b.length;                                // 5
b.toString();                            // 'hello'
b.toString('hex');                       // '68656c6c6f'
b.toString('base64');                    // 'aGVsbG8='

const out = Buffer.alloc(10);            // zero-filled
out.write('hi', 0, 'utf8');
```

Buffer extends `Uint8Array`, so you can pass one where a typed array is expected.

### Use case: building a binary response

```js
const header = Buffer.alloc(4);
header.writeUInt32BE(body.length, 0);
const packet = Buffer.concat([header, body]);
socket.write(packet);
```

### Watch out

- `Buffer.allocUnsafe(n)` skips zero-filling — **fast but can leak old memory content** if you don't overwrite it. Use only when you're about to fill the whole buffer.
- `Buffer.from('...')` without an encoding defaults to UTF-8, which *usually* is right, but not for ASCII-only hot paths.
- Buffer memory counts as `external` in `process.memoryUsage`, not `heapUsed` — a memory leak in buffers hides from heap snapshots unless you look at the external field.

---

## Chapter 9. Streams

### What it is

Streams are Node's abstraction for sequences of data handled **chunk by chunk** instead of all at once. Four types: **Readable** (source), **Writable** (sink), **Duplex** (both), **Transform** (duplex where output is derived from input).

### Why you care

Loading a 2 GB file with `fs.readFile` blows up your heap. `createReadStream` processes it in 64 KB pieces at constant memory. HTTP, TCP, compression, crypto, file I/O — all streams underneath.

### The modes

- **Flowing** (`readable.on('data', ...)` or `.pipe(...)`) — data is pushed as it arrives.
- **Paused** (`readable.read()`) — you pull when ready.
- **Object mode** — chunks are JS objects instead of bytes.

### How it works

```js
import { createReadStream, createWriteStream } from 'node:fs';
import { pipeline } from 'node:stream/promises';
import { createGzip } from 'node:zlib';

await pipeline(
  createReadStream('input.log'),
  createGzip(),
  createWriteStream('input.log.gz'),
);
```

`pipeline` propagates errors, cleans up resources, and returns a Promise — always prefer it over `.pipe()` chains.

### Backpressure — the point of streams

When the destination is slow, `write()` returns `false`. You **stop writing**, wait for `'drain'`, then continue. `pipeline` handles this for you; with manual code, you must.

```js
function writeAll(stream, items) {
  for (const item of items) {
    if (!stream.write(item)) {
      return new Promise(resolve => stream.once('drain', resolve))
        .then(() => writeAll(stream, items.slice(items.indexOf(item) + 1)));
    }
  }
  stream.end();
}
```

### Transform streams — your own pipeline stage

```js
import { Transform } from 'node:stream';

const upper = new Transform({
  transform(chunk, _enc, cb) { cb(null, chunk.toString().toUpperCase()); }
});

await pipeline(process.stdin, upper, process.stdout);
```

### Watch out

- Mixing `.pipe()` and `'data'` listeners gives you duplicated reads.
- Not handling errors on a stream crashes the process on `unpipe`/`close`.
- Object-mode streams ignore the 64 KB highWaterMark — the unit is 1 object, so a high-volume pipeline needs custom throttling.

---

## Chapter 10. File System

### What it is

Three flavors: callback (`fs`), Promise (`fs/promises`), and synchronous (`fs.*Sync`). Underneath, libuv does the work on the thread pool.

### Examples

```js
import { readFile, writeFile, readdir, stat } from 'node:fs/promises';

const data = await readFile('config.json', 'utf8');
const parsed = JSON.parse(data);

await writeFile('out.txt', 'hello', 'utf8');
const names = await readdir('.');
const s = await stat('file.log');
console.log(s.size, s.mtime);
```

### Use case: streaming a huge file

```js
import { createReadStream } from 'node:fs';
import { createInterface } from 'node:readline';

const rl = createInterface({ input: createReadStream('big.log') });
for await (const line of rl) {
  if (line.includes('ERROR')) console.log(line);
}
```

### Watch out

- Never use `fs.*Sync` in a server — it blocks the event loop for the duration of the disk I/O. Fine for boot/config; never for per-request work.
- File system is on libuv's thread pool. CPU-bound crypto contending for the same pool can stall `fs` calls; bump `UV_THREADPOOL_SIZE` if needed.
- Paths are OS-specific. Always use `path.join` / `path.resolve` — never concatenate strings with `/`.

---

## Chapter 11. HTTP and HTTPS

### What it is

Node's built-in HTTP server and client, built on raw TCP. Express, Fastify, and every framework wrap this — understanding it pays off when things get weird.

### A minimal server

```js
import { createServer } from 'node:http';

const server = createServer(async (req, res) => {
  if (req.method === 'POST' && req.url === '/echo') {
    const chunks = [];
    for await (const chunk of req) chunks.push(chunk);
    const body = Buffer.concat(chunks).toString('utf8');
    res.writeHead(200, { 'content-type': 'application/json' });
    res.end(JSON.stringify({ body }));
    return;
  }
  res.writeHead(404); res.end('not found');
});

server.listen(3000);
```

### A minimal client (modern Node)

```js
const r = await fetch('https://api.example.com/users', {
  method: 'POST',
  headers: { 'content-type': 'application/json' },
  body: JSON.stringify({ name: 'Alice' }),
});
const data = await r.json();
```

### Keep-alive

```js
import { Agent } from 'node:https';
const agent = new Agent({ keepAlive: true, maxSockets: 100 });
// pass as { agent } to fetch/https.request
```

Reusing sockets saves TCP handshakes and TLS renegotiation — essential for high-throughput clients.

### Watch out

- `req.on('data')` chunks can split JSON mid-byte. Buffer fully before parsing or use a framework.
- `res.end()` must always be called — or the connection hangs until the client times out.
- Default `maxSockets` for the global agent is `Infinity` — you can exhaust file descriptors under load. Configure agents.

---

## Chapter 12. Net — Raw TCP and Unix Sockets

### What it is

`net.createServer` gives you a raw TCP server; `net.connect` gives you a client. HTTP is built on top of this.

### Use case: a simple protocol

```js
import { createServer } from 'node:net';

createServer((sock) => {
  sock.on('data', (buf) => sock.write(buf));   // echo
  sock.on('end', () => console.log('client gone'));
}).listen(9000);
```

### Watch out

- TCP is a byte stream, not messages. You need your own framing (length prefix, delimiter, or a protocol like HTTP/MQTT/Protobuf-over-TCP).
- Unix domain sockets (`net.createServer().listen('/tmp/sock')`) are faster than localhost TCP — use them between co-located processes (e.g., Nginx ↔ Node, sidecars).

---

# Part IV — Concurrency

## Chapter 13. Child Processes

### What it is

Four ways to run another program:

- `spawn(cmd, args)` — long-running, streams stdout/stderr, low overhead.
- `exec(cmd)` — buffers output, convenient for quick shell commands, **shell injection risk if you interpolate untrusted input**.
- `execFile(file, args)` — like exec but no shell.
- `fork(modulePath)` — spawns another Node process and sets up an IPC channel.

### Examples

```js
import { spawn, fork } from 'node:child_process';

const ls = spawn('ls', ['-la', '/tmp']);
for await (const chunk of ls.stdout) process.stdout.write(chunk);
const code = await new Promise(r => ls.on('close', r));
```

```js
// parent.js
const child = fork('./worker.js');
child.send({ cmd: 'compute', input: 42 });
child.on('message', (m) => console.log('result', m));
```

### Use case: running an untrusted tool (image resize, ffmpeg)

```js
const p = spawn('ffmpeg', ['-i', input, '-vf', 'scale=640:-1', output]);
```

### Watch out

- `exec` with user input is a textbook shell injection. Use `execFile` or `spawn` with argument arrays.
- Children don't auto-die when the parent crashes. Use `detached: false` and kill them in shutdown handlers.
- Large stdout can deadlock if you never read it — the pipe fills and the child blocks.

---

## Chapter 14. Cluster

### What it is

A module that lets a primary process fork multiple worker processes, each a copy of your Node app, all sharing the same listening socket.

### Why it exists

Node is single-threaded per process. To use multiple CPU cores on one machine, you run multiple processes. Cluster makes that easy — the OS handles load balancing across workers.

### How it works

```js
import cluster from 'node:cluster';
import os from 'node:os';

if (cluster.isPrimary) {
  for (let i = 0; i < os.cpus().length; i++) cluster.fork();
  cluster.on('exit', (w) => { console.log('worker died, restart'); cluster.fork(); });
} else {
  // same code path as a single-process server
  import('./server.js');
}
```

### Use case vs PM2 vs Kubernetes

- **Cluster** — single-machine, quick, no extra ops.
- **PM2** — cluster + restart + log management + zero-downtime reload, one CLI.
- **Kubernetes / ECS** — treat each pod as a single Node process; let the orchestrator do the scaling. Modern preferred path.

### Watch out

- Workers share nothing but a listening port. Per-worker in-memory caches are per-worker — use Redis for shared state.
- Sticky sessions (WebSocket) need session affinity; cluster round-robins by default.

---

## Chapter 15. Worker Threads

### What it is

Real OS threads, each with its own V8 instance, event loop, and memory. They talk via `postMessage` (structured clone) or `SharedArrayBuffer` + `Atomics`.

### Why you care

Worker Threads are how you run CPU-bound work (image processing, data crunching, regex) without blocking the main event loop. **Cluster spawns processes; Worker Threads are threads.**

### Example

```js
// main.js
import { Worker } from 'node:worker_threads';

const w = new Worker('./heavy.js', { workerData: { n: 40 } });
w.on('message', (result) => console.log('got', result));
w.on('error', console.error);
w.on('exit', (code) => console.log('exit', code));
```

```js
// heavy.js
import { parentPort, workerData } from 'node:worker_threads';
function fib(n) { return n < 2 ? n : fib(n-1) + fib(n-2); }
parentPort.postMessage(fib(workerData.n));
```

### Use case: a worker pool

Workers are expensive to start (~50–100 ms). For short jobs, keep a pool:

```js
import { Worker } from 'node:worker_threads';
class Pool {
  constructor(path, size) {
    this.idle = [];
    this.queue = [];
    for (let i = 0; i < size; i++) this.idle.push(new Worker(path));
  }
  run(task) {
    return new Promise((resolve, reject) => {
      const go = (w) => {
        w.once('message', resolve);
        w.once('error', reject);
        w.postMessage(task);
      };
      const w = this.idle.shift();
      if (w) go(w);
      else this.queue.push(go);
    });
  }
}
```

(Production pools add error recovery, timeouts, worker recycling. See `piscina`.)

### Watch out

- No shared memory except `SharedArrayBuffer`. All other values are structured-cloned on `postMessage` (copies — not free).
- Don't use threads for I/O — libuv already handles that. Threads are for **CPU work**.

---

## Chapter 16. AsyncLocalStorage

### What it is

A way to attach per-request context to *every* async callback spawned during that request, without passing it as a parameter. Implemented on top of async_hooks.

### Why you care

Request IDs, user identity, tenant IDs in logs — you need them everywhere. Passing them through every function signature is noisy; `als.run()` scopes them automatically to all async descendants.

### Example

```js
import { AsyncLocalStorage } from 'node:async_hooks';

const als = new AsyncLocalStorage();

function log(msg) {
  const store = als.getStore();
  console.log(`[${store?.requestId ?? '-'}] ${msg}`);
}

app.use((req, res, next) => {
  als.run({ requestId: req.headers['x-request-id'] || crypto.randomUUID() }, next);
});

app.get('/x', async (req, res) => {
  log('handling');                     // logs with the right requestId
  await doWork();                      // doWork can also call log() and see the same id
  res.end();
});
```

### Watch out

- `als.run()` creates a new scope — everything async inside it shares the store. Outside, the store is gone.
- Older Node had measurable overhead for async_hooks; recent versions have it down to near-zero.
- Some third-party thread-bound code (e.g., native addons with their own event loops) may break propagation. Test before relying on it in hot paths.

---

# Part V — Production Concerns

## Chapter 17. Error Handling Patterns

### What it is

Two classes of errors with very different handling:

- **Operational errors** — expected, recoverable: bad input, 404, timeout, disk full. Handle per-request, return an error response, keep running.
- **Programmer errors** — bugs, invariant violations: null dereference, wrong types, broken state machine. **Crash and restart.**

### Why the distinction matters

Trying to "handle" a programmer error leaves the process in an unknown state. Subsequent requests see weird, hard-to-debug failures. Crashing is the safe choice; a supervisor restarts.

### Patterns

```js
// 1. Custom error classes with codes
class NotFoundError extends Error {
  constructor(msg) { super(msg); this.name = 'NotFoundError'; this.code = 'E_NOT_FOUND'; }
}

// 2. Central async handler (Express pattern)
app.use(async (err, req, res, next) => {
  if (res.headersSent) return next(err);
  const status = err.status || (err.code === 'E_NOT_FOUND' ? 404 : 500);
  req.log.error({ err });
  res.status(status).json({ error: err.message });
});

// 3. Process-wide safety nets
process.on('uncaughtException', (err) => { logger.fatal({ err }); process.exit(1); });
process.on('unhandledRejection', (err) => { logger.fatal({ err }); process.exit(1); });
```

### Watch out

- `throw` inside an EventEmitter listener is **not** caught — it propagates to `uncaughtException`.
- Never `process.exit(1)` inside a library — let callers decide.
- Rejected promises in a for-loop are not awaited if you forget `await` — silent failures.

---

## Chapter 18. Graceful Shutdown

### What it is

When you receive `SIGTERM` (Kubernetes sends this before `SIGKILL`), stop accepting new work, finish what's in flight, close DBs and caches, and exit with 0.

### Why you care

Ungraceful shutdowns drop requests, leave transactions half-committed, and corrupt logs. Every serious service needs a 30-second drain window.

### Pattern

```js
let shuttingDown = false;

const server = app.listen(port);

async function shutdown(signal) {
  if (shuttingDown) return;
  shuttingDown = true;
  logger.info({ signal }, 'shutting down');

  // 1. Stop accepting new HTTP connections
  server.close(() => logger.info('http closed'));

  // 2. Drain in-flight (wait up to N seconds)
  await drainInFlight(30_000);

  // 3. Close external resources
  await db.destroy();
  await redis.quit();

  process.exit(0);
}

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT',  () => shutdown('SIGINT'));
```

### Watch out

- `server.close()` waits for existing keep-alive connections; destroy idle sockets yourself or they'll keep the process alive:
  ```js
  server.closeIdleConnections();     // Node 18.2+
  ```
- Load balancers need a moment to notice you're gone. Many teams add `/healthz` that returns 503 during shutdown, then actually close a few seconds later.

---

## Chapter 19. Logging in Production

### What it is

Structured logs (JSON lines, key/value) — never free-form text in production. One line per event, with request id, level, timestamp, and message.

### Why you care

Splunk, Datadog, CloudWatch, Loki all love JSON. Free-form logs are grep-only, require regex, and break under pressure.

### Example with `pino`

```js
import pino from 'pino';
const log = pino({ level: process.env.LOG_LEVEL || 'info' });

log.info({ userId, route: req.path }, 'request');
log.error({ err }, 'failed');
```

Output:
```json
{"level":30,"time":1713619200000,"msg":"request","userId":42,"route":"/x"}
```

### Use case: request-scoped logging with AsyncLocalStorage

Combine Chapter 16's ALS with pino's child loggers:

```js
const log = pino();
const als = new AsyncLocalStorage();

app.use((req, res, next) => {
  const reqLog = log.child({ requestId: req.headers['x-request-id'] });
  als.run({ log: reqLog }, next);
});

// anywhere downstream:
als.getStore().log.info('processing');
```

### Watch out

- Log volume costs money. Keep `debug`/`trace` off in prod; log expensive payloads behind a level check (`log.isLevelEnabled('debug') && log.debug(...)`).
- Never log secrets, tokens, or full request bodies by default. Use redaction config.

---

## Chapter 20. Configuration

### What it is

Environment variables — not hardcoded values, not files committed to git. Validated at boot, typed, fail-fast if missing.

### Pattern

```js
import { z } from 'zod';

const Env = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
});

export const env = Env.parse(process.env);
// Typed, validated, and available everywhere via `import { env } from './env'`.
```

### Watch out

- Don't read `process.env.X` scattered across the codebase — boot-time validation is the whole point.
- Secrets belong in a secret manager (AWS Secrets Manager, Vault), not `.env` files committed by accident. Add `.env` to `.gitignore` forever.

---

# Part VI — Advanced Runtime

## Chapter 21. Async Hooks (briefly)

### What it is

A low-level API that lets you observe every async resource lifecycle: created, before-callback, after-callback, destroyed. `AsyncLocalStorage` is built on top of it.

### Why you care (a little)

You almost never use async_hooks directly. It's the foundation for tracing (OpenTelemetry), profilers, and `AsyncLocalStorage`. Know it exists; reach for `AsyncLocalStorage` 99% of the time.

### Example

```js
import async_hooks from 'node:async_hooks';
const hook = async_hooks.createHook({
  init(asyncId, type, triggerAsyncId) { /* ... */ },
  destroy(asyncId) { /* ... */ },
});
hook.enable();
```

### Watch out

- Enabling async_hooks has overhead — historically ~10–20%, now much lower but still non-zero. Don't enable if you don't need it.

---

## Chapter 22. Crypto

### What it is

Built-in bindings to OpenSSL: hashing, HMAC, symmetric/asymmetric ciphers, key derivation, random bytes, TLS. Plus a web-compatible `crypto.webcrypto` interface.

### Why you care

Every auth system, session signer, webhook verifier, encryption-at-rest implementation goes through this. Rolling your own is a security hole.

### Common tasks

```js
import { randomBytes, createHash, createHmac, timingSafeEqual, scrypt } from 'node:crypto';

// Random token
const tok = randomBytes(32).toString('hex');

// Hash (non-password!): SHA-256
createHash('sha256').update('hello').digest('hex');

// HMAC for signatures
const sig = createHmac('sha256', secret).update(payload).digest('hex');

// Constant-time compare (don't use === for signatures)
const ok = timingSafeEqual(Buffer.from(a, 'hex'), Buffer.from(b, 'hex'));

// Password hashing — scrypt (preferred) or argon2 via library
const key = await new Promise((resolve, reject) =>
  scrypt('password', 'salt', 64, (err, k) => err ? reject(err) : resolve(k))
);
```

### Watch out

- **Never hash passwords with SHA-256.** Use scrypt, argon2, or bcrypt with a salt per user.
- Use `timingSafeEqual` for comparing secrets, HMACs, tokens — `a === b` leaks timing.
- Crypto operations run on libuv's thread pool. Bulk crypto can starve `fs`; size your pool (`UV_THREADPOOL_SIZE`) accordingly.

---

## Chapter 23. Debugging and Profiling

### What it is

Node has a built-in V8 inspector. Run with `--inspect` and connect Chrome DevTools, VS Code, or the Node CLI debugger.

### CLI flags worth knowing

```
--inspect                attach DevTools (non-blocking start)
--inspect-brk            break on first line (handy for entry-point bugs)
--prof                   sample CPU → isolate-*.log → process with --prof-process
--cpu-prof               write .cpuprofile (open in DevTools)
--heap-prof              sample heap allocations
--trace-warnings         stack traces on warnings
--trace-deprecation      stack traces on deprecations
--unhandled-rejections=strict    crash on unhandled rejections (default in new Node)
```

### Clinic.js — higher-level tooling

```
npx clinic doctor  -- node server.js   # detect event loop lag / I/O issues
npx clinic flame   -- node server.js   # flame graph of on-CPU time
npx clinic heap    -- node server.js   # heap profiling
```

### Event loop lag monitoring

```js
import { monitorEventLoopDelay } from 'node:perf_hooks';
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => {
  const p99ms = h.percentile(99) / 1e6;
  if (p99ms > 100) logger.warn({ p99ms }, 'event loop lagging');
  h.reset();
}, 5000);
```

### Watch out

- Profiling adds overhead. Don't leave `--cpu-prof` on in production unless you're collecting with sampling and rate-limiting.
- A single hot sync function (JSON.parse of a huge body, a regex backtrack) can cause 100 ms+ loop lag — always measure with monitorEventLoopDelay in prod.

---

## Chapter 24. Native Addons and N-API

### What it is

C/C++ code compiled into a shared library and loaded into Node via `require`. Used to wrap OS APIs, expose native libraries (like image codecs, sqlite, v8 internals), or escape V8 for performance.

### Why you care (a little)

You'll *use* native addons (bcrypt, sharp, better-sqlite3) often. You'll *write* one rarely. If you do, you'll use **N-API** (the stable ABI) via `node-addon-api` (C++) or `napi-rs` (Rust).

### Tradeoffs

- Pro: access to native code, faster than JS for tight numeric loops.
- Con: platform builds (prebuilt binaries or node-gyp), harder to debug, can crash the process, slower to start (dynamic linking).

### Watch out

- A native crash kills Node with no JS stack trace. Use sanitizers while developing.
- Prebuilt binaries via `prebuild` / `node-pre-gyp` are the difference between "npm install works" and "users yell at you on Windows".

---

# Appendix — Node.js Production Checklist

- [ ] Env vars validated at boot (zod/envalid); no `process.env.X` scattered around.
- [ ] Structured JSON logs with a requestId per request (ALS).
- [ ] `SIGTERM`/`SIGINT` handler with drain + resource close + `process.exit(0)`.
- [ ] `uncaughtException` / `unhandledRejection` handlers that log fatally and exit.
- [ ] `server.keepAliveTimeout` and `headersTimeout` set above your LB's idle timeout.
- [ ] `UV_THREADPOOL_SIZE` tuned if you do heavy crypto/fs/DNS.
- [ ] Event loop delay monitored and alerted on (> 100 ms p99).
- [ ] Graceful startup: readiness probe returns 200 only after DB/cache connected.
- [ ] Graceful shutdown: liveness returns 503 during drain.
- [ ] Security headers (helmet), CORS pinned, CSRF on cookie auth, rate limit.
- [ ] Secrets in a manager, not `.env` in git.
- [ ] Dependabot / `npm audit` in CI.
- [ ] Docker image: non-root user, minimal base (distroless/alpine), init process (`--init` or `tini`).
- [ ] Tracing and metrics (OpenTelemetry → Datadog/Grafana).

---

## Afterword

Node.js rewards engineers who stop treating it as "JavaScript on the server" and start treating it as "V8 + libuv + a carefully chosen API surface". Everything unusual about Node — event loop phases, backpressure, `uncaughtException`, cluster vs workers — maps cleanly to that architecture once you can see it.

Read the libuv book. Read the Node changelog. Turn on `monitorEventLoopDelay`. Your future self will thank you.
