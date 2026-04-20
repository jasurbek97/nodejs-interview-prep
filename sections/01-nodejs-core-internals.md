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

