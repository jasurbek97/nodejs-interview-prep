## 18. Deep Dive — Node.js & V8 Internals

### Q149. How does V8 compile and optimize JavaScript?

V8 is a **multi-tier JIT**:

1. **Parser** → AST.
2. **Ignition** — bytecode interpreter. Runs the code immediately and gathers type feedback (which types each operation actually sees).
3. **Sparkplug** — non-optimizing baseline compiler that turns bytecode into machine code very quickly. No optimizations, just a speed boost over the interpreter.
4. **Maglev** (newer mid-tier) — fast optimizing compiler for "hot enough" code.
5. **TurboFan** — top-tier optimizer. Speculatively inlines, eliminates dead code, unboxes numbers, uses inline caches. Produces highly optimized machine code based on the type feedback.

If a speculative assumption is violated (e.g., a function that always saw integers suddenly sees a string), V8 **deoptimizes** back to bytecode and may re-optimize later.

**Why it matters:** keep functions monomorphic (one shape of input) for as long as possible — that's what TurboFan rewards.

### Q150. Hidden classes and inline caches

V8 doesn't store JS objects as hashmaps. It assigns each object a **hidden class** (a.k.a. shape/map) describing its property layout. Two objects with the same properties added in the same order share a hidden class.

```js
function Point(x, y) { this.x = x; this.y = y; } // shape A → B → C

const a = new Point(1, 2);
const b = new Point(3, 4);  // same hidden class as a
b.z = 5;                     // b transitions to a NEW hidden class
```

**Inline caches (ICs)** at each property access remember the hidden class they last saw, so future accesses become a pointer lookup at a fixed offset.

- **Monomorphic IC** (1 shape) — fastest.
- **Polymorphic** (2–4 shapes) — fast-ish.
- **Megamorphic** (>4) — falls back to a generic dictionary lookup. Avoid.

**Practical rules:**
- Always initialize all properties in the constructor, in the same order.
- Avoid `delete obj.prop` — converts the object to dictionary mode.
- Don't add properties later in the lifetime.

### Q151. V8's heap layout and garbage collection

The V8 heap is split into **generations**:

| Space | What | GC algorithm |
|---|---|---|
| **New space** (young) | Newly allocated objects, ~1–8 MB | **Scavenge** (Cheney's copying GC) — very fast, runs often |
| **Old space** | Objects that survived 2 scavenges | **Mark-sweep-compact** with **incremental + concurrent** marking |
| **Large object space** | Objects ≥ ~512 KB | Allocated directly, swept in place |
| **Code space** | Compiled JIT code | |
| **Map space** | Hidden classes | |

Most objects die young (the **generational hypothesis**). Scavenge cleans them in microseconds. Survivors are promoted to old space, where collection is more expensive but rarer.

V8 uses **incremental marking** (small slices interleaved with JS execution) and **concurrent marking** (on a helper thread) to keep stop-the-world pauses to a few ms even on multi-GB heaps.

Tune with:
- `--max-old-space-size=4096` (MB) — raise the old-space cap.
- `--max-semi-space-size` — young-space size; bigger means fewer scavenges but longer ones.
- `--expose-gc` + `global.gc()` for tests.

### Q152. Common V8 optimization killers

- **`try/catch` around hot loops** — historically prevented optimization (much better since V8 ~6.0, but still costs).
- **`with` statement and `eval` with non-strict scope** — deoptimize aggressively.
- **`arguments` object leaks** — passing `arguments` to another function. Use rest parameters: `(...args) => ...`.
- **Mixing types in one variable** — `let x = 1; x = 'hi';`.
- **Polymorphic functions** — same function called with too many object shapes.
- **`delete obj.prop`** — pushes objects into dictionary mode.
- **Sparse arrays / holes** — `arr[1000] = 1` on an empty array creates a holey array, slower than packed.
- **Constructor-less property addition** — adding properties after construction creates new hidden class transitions for every instance.

### Q153. libuv thread pool — when does it matter?

libuv has a default pool of **4 threads** (cap 1024). These handle:

- File I/O (`fs.*`)
- DNS (`dns.lookup` — the synchronous resolver. `dns.resolve` uses async network.)
- Some `crypto` ops (`pbkdf2`, `randomBytes`, `scrypt`)
- `zlib` (gzip/deflate)
- Some `dgram` ops

**Symptom of saturation:** under load, file/crypto requests queue, latency rises but CPU is low. Tune with:

```bash
UV_THREADPOOL_SIZE=16 node server.js
```

Network I/O (TCP, UDP) does **not** use the pool — it's fully async via epoll/kqueue/IOCP, polled in the event loop's poll phase.

Rule: bump the pool only if you have measured pool starvation (`async_hooks` or `uv_metrics_info` in newer Node).

### Q154. The event loop's "ref count" — when does Node exit?

The event loop keeps running while there's **at least one referenced handle** (timer, socket, server, etc.) or pending request. Node exits when the count hits zero.

```js
const t = setTimeout(() => {}, 60_000);
t.unref();   // doesn't keep the loop alive
t.ref();     // does
```

`server.unref()` lets you keep a debug HTTP server alive only while there's other work. Same on streams, sockets, child processes. This is what lets a CLI script exit cleanly without manually tearing things down.

### Q155. AsyncLocalStorage — what is it and what does it cost?

`AsyncLocalStorage` (ALS) is Node's built-in mechanism for **per-request context propagation** across async boundaries. It uses `async_hooks` under the hood to carry a context object through promises and callbacks without passing it explicitly.

```js
import { AsyncLocalStorage } from 'node:async_hooks';
const als = new AsyncLocalStorage();

app.use((req, res, next) => {
  als.run({ requestId: req.headers['x-request-id'] }, next);
});

function log(msg) {
  const ctx = als.getStore();
  console.log(`[${ctx?.requestId}] ${msg}`);
}
```

**Use cases:** request IDs, tenant context, transaction objects, OpenTelemetry spans.

**Cost:** modest in modern Node (post-19, native AsyncContextFrame implementation). Each promise chain creation has a small overhead. Don't enable global `async_hooks` (different API) — that's the slow one.

### Q156. Production diagnostics toolkit

| Symptom | Tool |
|---|---|
| High CPU | `node --prof` then `--prof-process`; `0x` for flamegraphs; clinic.js `flame` |
| Slow event loop | clinic.js `doctor` / `bubbleprof`; `perf_hooks.monitorEventLoopDelay()` |
| Memory leak | `node --inspect` + Chrome DevTools heap snapshots; diff three snapshots; `heapdump` module |
| Native memory leak | `mimalloc`/`jemalloc` allocator + `MALLOC_CONF`; `valgrind` (Linux); `--heap-prof` |
| GC pauses | `--trace-gc`, `--trace-gc-verbose`; `perf_hooks.PerformanceObserver` on `'gc'` |
| Async stalls | `async_hooks` + clinic `bubbleprof`; OpenTelemetry traces |
| Crash on prod | `--report-on-fatalerror` writes `report-*.json` with stack/heap stats |
| Live debugging | `node --inspect=0.0.0.0:9229` + Chrome DevTools (gate by SSH tunnel) |

Always take **3 heap snapshots** during a leak investigation — between snapshot 1 and 2 you exercise the suspect path, between 2 and 3 you let it settle. Objects that grow across both intervals are leaking.

### Q157. Native addons — N-API in 60 seconds

Three options, in order of preference:

1. **Node-API (N-API / `node-addon-api`)** — ABI-stable C/C++ API. The addon you compile against Node 20 keeps working on Node 22 without recompiling. **Default choice.**
2. **NAN** — older, requires recompile per Node major version. Legacy only.
3. **Internal V8 API** — fastest but breaks every release. Don't.

Build with `node-gyp` or `cmake-js`. For Rust, **napi-rs** generates N-API bindings and is what most modern native addons use (e.g., SWC, Parcel).

When to reach for native: CPU-heavy hot paths (image processing, hashing, parsing) where worker_threads + JS isn't fast enough, or wrapping an existing C/C++ library.

### Q158. WebAssembly in Node

Node ships with V8's WebAssembly engine. You can:

```js
const wasm = await WebAssembly.instantiate(bytes, imports);
wasm.instance.exports.myFn(42);
```

**When wasm beats JS:** numerical loops, parsers, codecs, anything compiled from C/Rust/Zig. **WASI** (`node --experimental-wasi-unstable-preview1`) lets wasm modules touch the filesystem.

**Trade-offs:** copying data across the JS/wasm boundary is the bottleneck. Use shared `ArrayBuffer` (`memory.buffer`) views, not per-call copies.

For most Node web servers, wasm is overkill — but for CPU-bound services (image transforms, regex-heavy parsers, ML inference) it's a serious lever.

---

