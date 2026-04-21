## 26. Deep Dive — JavaScript Internals

Section 2 covered the everyday JavaScript a senior should have fluent. This deep dive is what separates "I use JS" from "I know what the engine is doing". Expect these in staff/principal interviews or when debugging something a stack trace alone can't explain.

### Q249. The execution model — contexts, scope chain, lexical environment

Every function call creates an **execution context** pushed onto the call stack. A context has:

- A **LexicalEnvironment** — a record of identifiers declared in this scope plus a pointer to the parent (outer) environment. Forms the **scope chain**.
- A **VariableEnvironment** — historically for `var` (function-scoped); identical to LexicalEnvironment in modern code.
- A `this` binding.

Identifier lookup walks the chain outward. Closures exist because an inner function keeps a reference to its creation-time outer environment — so even after the outer function returns, its variables are reachable via the inner function's `[[Environment]]` slot, and are not collected.

```js
function make() {
  const secret = 42;                    // lives in make's lexical env
  return () => secret;                  // inner keeps [[Environment]] → make's env
}
const peek = make();                    // make's frame is gone from the stack,
peek();                                 // but its env is alive on the heap → 42
```

This is also why a naive closure over a loop variable with `var` captures the final value — there is only one binding in the enclosing function scope. `let` creates a fresh binding per iteration.

### Q250. What happens when V8 runs your code

V8's pipeline, roughly:

1. **Parser** → AST.
2. **Ignition** (bytecode interpreter) starts executing immediately. Cheap to start, slow per op.
3. **Sparkplug** (non-optimizing baseline JIT) compiles hot-ish bytecode to native code.
4. **Maglev** (mid-tier optimizing JIT, newer) and **TurboFan** (top-tier) kick in for truly hot functions. They speculate on types and inline aggressively.
5. **Deopt** — if a speculation is violated (e.g., a function that was always called with numbers is suddenly called with a string), the optimized code is thrown out and execution falls back to Ignition.

Two concepts that matter for performance:

- **Hidden classes (shapes / maps)** — V8 gives each object a hidden class based on its property layout. Objects that get the same properties in the same order share a class, and property access becomes a fixed offset load. Adding properties in different orders, or deleting properties, forks the shape and slows everything down.
- **Inline caches (ICs)** — at each property access / call site, V8 remembers the shape it saw last. A site is *monomorphic* (one shape — fast), *polymorphic* (2–4 — still fast), or *megamorphic* (5+ — fall back to a dictionary lookup). Code that sees too many shapes at a hot site silently loses 10–100×.

Practical rules: initialize all properties in a constructor in a fixed order, avoid `delete` on hot objects, don't mix shapes at a hot call site, don't monkey-patch prototypes after the app has warmed up.

### Q251. The four equality algorithms

Spec-level, JS has four:

| Algorithm | Used by | `NaN === NaN` | `+0 === -0` |
|---|---|---|---|
| **Strict Equality** (`===`) | `===`, `switch` | false | true |
| **Abstract Equality** (`==`) | `==` | false | true |
| **SameValueZero** | `Array.includes`, `Map`/`Set` keys | **true** | true |
| **SameValue** | `Object.is` | **true** | **false** |

```js
[NaN].includes(NaN);                 // true  (SameValueZero)
new Map([[NaN, 1]]).get(NaN);        // 1     (SameValueZero)
NaN === NaN;                         // false (Strict)
Object.is(+0, -0);                   // false (SameValue)
```

Abstract equality (`==`) applies a long type-coercion ladder (ToPrimitive → ToNumber / ToString). Memorable cases: `null == undefined` (true), `null == 0` (false — `null` doesn't coerce to number for `==`), `'' == 0` (true), `[] == false` (true).

### Q252. Memory model and garbage collection

JS has two regions:

- **Stack** — fixed-size frames holding primitives and pointers. Freed when the frame pops.
- **Heap** — objects, closures, arrays, strings (mostly). Managed by the GC.

V8 uses a **generational, incremental, mostly-concurrent** GC:

- **Young generation** (new space) — small, collected frequently with a copying **scavenger**. Survivors are promoted.
- **Old generation** — collected with **Mark-Sweep** and occasionally **Mark-Compact**. Large, infrequent.
- **Incremental marking** — marking is done in small slices interleaved with JS execution so pauses stay sub-millisecond on well-behaved code.
- **Concurrent / parallel** phases run on helper threads.

Common leaks:
- Unbounded caches (use an LRU).
- Listeners attached and never removed.
- Closures that capture large objects even though they only need one field — V8 can't always prune.
- Detached DOM nodes still referenced from JS (browser).
- `setInterval` callbacks referencing request-scoped data.

Useful tools: Chrome DevTools heap snapshot (find retainers), `--inspect` + Node's `--heap-prof`, `--max-old-space-size` to raise the heap limit when you've *verified* it's real usage not a leak.

### Q253. Weak references — WeakMap, WeakSet, WeakRef, FinalizationRegistry

`Map`/`Set` keep their keys alive. `WeakMap`/`WeakSet` hold their **keys weakly** — if no other reference exists, the entry is GC'd.

```js
const cache = new WeakMap();
function tagged(obj) {
  if (!cache.has(obj)) cache.set(obj, compute(obj));
  return cache.get(obj);
}
// When `obj` is no longer referenced elsewhere, the cache entry disappears automatically.
```

Constraints: WeakMap keys must be objects (or symbols in recent specs), not primitives. WeakMap is not iterable — you can't enumerate or get a size, because that would expose GC timing.

`WeakRef` gives you a hold-on-if-still-alive handle to any object; `ref.deref()` returns the object or `undefined`.

`FinalizationRegistry` lets you run a callback after an object is collected. **Avoid unless you really need it** — timing is non-deterministic, and finalizers run late. Real uses: freeing off-heap resources (native handles in an addon), logging leaks in dev.

### Q254. Promise internals — state, queue, and the three traps

A Promise is a state machine with three states (pending → fulfilled | rejected) and a FIFO list of reactions (`.then`/`.catch`/`.finally` handlers).

When `resolve(v)` is called:
1. If `v` is a thenable, `resolve` adopts its state — `Promise.resolve(Promise.resolve(x))` flattens.
2. Otherwise state transitions to fulfilled.
3. All registered reactions are scheduled as **microtasks** on the current job queue.

Microtasks drain after each macrotask completes and between every "tick" of the event loop. They run to completion before any I/O or timer fires. This is why a tight promise chain can starve I/O — but also why `await` is synchronous-feeling.

Three traps:

```js
// 1. Unhandled rejection — no .catch, no await that throws → process event 'unhandledRejection'
Promise.reject(new Error('x'));

// 2. Forgotten await silently swallows errors
async function run() {
  fetchSomething();                    // returns a rejected promise, no one is listening
}

// 3. Returning vs awaiting inside a try/catch
async function outer() {
  try { return inner(); }              // DOES NOT catch — caller sees the rejection
  catch (e) { /* never runs */ }
}
async function outerFixed() {
  try { return await inner(); }        // catches
  catch (e) { /* runs */ }
}
```

Rule of thumb: `await` at the boundary where you want to handle errors; never leave a floating promise unless you explicitly `.catch()` it.

### Q255. What `async/await` actually desugars to

Roughly, an `async function` is a generator whose `yield` is `await`, driven by a runner that threads Promise resolutions back in:

```js
async function f() {
  const a = await p1;
  const b = await p2(a);
  return a + b;
}

// ≈
function f() {
  return new Promise((resolve, reject) => {
    const gen = (function*() {
      const a = yield p1;
      const b = yield p2(a);
      return a + b;
    })();
    function step(val, isErr) {
      let r;
      try { r = isErr ? gen.throw(val) : gen.next(val); }
      catch (e) { return reject(e); }
      if (r.done) return resolve(r.value);
      Promise.resolve(r.value).then(v => step(v), e => step(e, true));
    }
    step();
  });
}
```

Implications: each `await` is a microtask boundary — state is preserved on the heap, the stack unwinds, and resumption is scheduled. Two `await`s in sequence cost two microtask turns even if both promises are already resolved.

### Q256. ESM vs CommonJS — the internals that actually matter

| | CommonJS | ESM |
|---|---|---|
| Loader | Sync, on demand | Async, parsed before execution |
| Bindings | **Copy** of values at `require` time | **Live** bindings (the import sees the latest exported value) |
| `this` at top level | `module.exports` | `undefined` |
| Circular deps | Returns a partial `exports` at the cycle point | Works via live bindings, but TDZ errors if you use the import before it's initialized |
| Top-level `await` | Not supported | Supported; makes the module async |
| File extensions | Optional | Required (except with `--experimental-specifier-resolution=node`) |
| JSON/CSS imports | `require('./x.json')` | Needs `assert { type: 'json' }` / import attributes |

Key gotcha — **default export shape**:
```js
// CJS
module.exports = { a: 1 };             // require returns the object

// ESM
export default { a: 1 };               // the *default* export IS the object
// From CJS: const m = require('./x.mjs');  m.default.a
// From ESM: import m from './x.js';         m.a
```

Interop: Node lets ESM `import` CJS (gets `module.exports` as `default`, and named exports when statically analyzable). CJS `require()`ing ESM is only supported for sync ESM modules in recent Node versions; otherwise use dynamic `import()`.

### Q257. Proxy and Reflect

A `Proxy` wraps a target object and intercepts fundamental operations via **traps**:

```js
const user = { name: 'Alice', age: 30 };
const guarded = new Proxy(user, {
  get(target, prop, receiver) {
    if (prop.startsWith('_')) throw new Error('private');
    return Reflect.get(target, prop, receiver);
  },
  set(target, prop, value) {
    if (prop === 'age' && typeof value !== 'number') throw new TypeError();
    return Reflect.set(target, prop, value);
  },
});
```

Traps: `get`, `set`, `has`, `deleteProperty`, `ownKeys`, `getOwnPropertyDescriptor`, `defineProperty`, `getPrototypeOf`, `setPrototypeOf`, `isExtensible`, `preventExtensions`, `apply`, `construct`.

`Reflect` mirrors those operations as functions — use it inside traps to forward to the default behavior with the right `receiver` (important when the target's getters use `this`).

**Invariants** are enforced by the engine: e.g., a `get` trap must return the same value as the target for non-writable, non-configurable data properties. Violations throw.

Real uses: reactive systems (Vue's reactivity), ORMs (Sequelize lazy loads), validation wrappers, test mocks. Cost: a proxy is ~10× slower than a raw object for hot access — wrap at boundaries, not in hot loops.

### Q258. Iteration protocols

An object is **iterable** if it has a `Symbol.iterator` method returning an **iterator**: an object with `next()` → `{ value, done }`.

```js
const range = {
  from: 1, to: 5,
  [Symbol.iterator]() {
    let i = this.from, end = this.to;
    return { next: () => i <= end ? { value: i++, done: false } : { value: undefined, done: true } };
  }
};
for (const n of range) console.log(n);
[...range];                            // spread uses the same protocol
```

Async iteration uses `Symbol.asyncIterator` and `next()` returning a promise. `for await (const x of src)` awaits each step.

Built-in iterables: Array, String (by code point — surrogate-pair safe), Map, Set, arguments, NodeList, TypedArrays, generators. Plain objects are **not** iterable — use `Object.entries(obj)` / `Object.keys(obj)`.

Generators (`function*`) are the ergonomic way to implement iterators — they also expose `return()` and `throw()`, which `for...of` uses to clean up on early break or error (e.g., closing a file handle). If you write manual iterators, handle these for parity.

### Q259. RegExp — the parts that bite

```js
// 1. Sticky (y) — matches at lastIndex only; great for tokenizers
const re = /\d+/y;
re.lastIndex = 5;
re.exec('abc  123');                   // null — no match AT 5

// 2. Unicode (u) — treats surrogate pairs as one code point, enables \p{} classes
/^.$/.test('𝐀');                       // false — counts surrogate halves
/^.$/u.test('𝐀');                      // true

// 3. Named groups + backrefs
'2026-04-20'.match(/(?<y>\d{4})-(?<m>\d{2})-(?<d>\d{2})/).groups; // {y, m, d}

// 4. Lookbehind (ES2018) — variable-length in V8
'price: $42'.match(/(?<=\$)\d+/);      // ['42']

// 5. matchAll — all matches with groups (replaces global exec loops)
[...'a1 b2 c3'.matchAll(/(\w)(\d)/g)];
```

Pitfalls: global flag `/g` on `.test()` / `.exec()` keeps `lastIndex` between calls — reuse a `/g` regex and you'll get surprising results; use `.matchAll` or a fresh regex each time. Catastrophic backtracking (e.g., `/(a+)+b/` on `'aaaa...'`) can hang the event loop — prefer atomic groups (via lookaheads) or write a manual parser for adversarial input.

### Q260. Numbers, precision, and BigInt

All `number` values are IEEE 754 double-precision floats. Consequences:

```js
0.1 + 0.2;                             // 0.30000000000000004
Number.MAX_SAFE_INTEGER;               // 2^53 - 1 = 9007199254740991
9007199254740993 === 9007199254740992; // true — beyond safe integer range
0.1 + 0.2 === 0.3;                     // false; use a tolerance
Math.abs((0.1 + 0.2) - 0.3) < Number.EPSILON; // true
```

For money: use integer minor units (cents) or a decimal library (`decimal.js`, `big.js`) — never floats. For IDs > 2^53: use strings or BigInt end-to-end (JSON.parse turns `9999999999999999` into `10000000000000000` — lossy).

`BigInt` (`1n`, `BigInt(x)`): arbitrary precision integers. Doesn't mix implicitly with Number (`1n + 1` throws). No `Math.*` support. JSON doesn't serialize it — you need a custom replacer/reviver.

### Q261. Modern classes — private fields, static blocks, and why

```js
class Counter {
  #count = 0;                          // true private, enforced by the engine
  static #registry = new Map();        // private static
  static {                             // static initialization block
    Counter.#registry.set('default', new Counter());
  }
  inc() { return ++this.#count; }
  static get(name) { return Counter.#registry.get(name); }
}
```

Differences from TS `private`:
- TS `private` is a compile-time hint — at runtime it's a regular property, visible and writable.
- `#private` is enforced by the engine. Accessing `obj.#x` from outside the class is a **SyntaxError**, not a runtime error — caught at parse time.
- `in` operator with private names is a branding check: `#count in obj` is true only for real instances. Useful in `Symbol.hasInstance` for nominal typing.

Class fields (including `#` ones) are installed in the constructor *after* `super()` but before the constructor body runs. Arrow-function fields (`onClick = () => ...`) are per-instance — they keep `this` bound but cost memory; prototype methods are shared.

### Q262. Typed arrays, SharedArrayBuffer, Atomics

Typed arrays (`Uint8Array`, `Int32Array`, `Float64Array`, ...) are views over an `ArrayBuffer` — contiguous raw bytes. Used for binary protocols, WebGL, crypto, streaming.

```js
const buf = new ArrayBuffer(16);
const u32 = new Uint32Array(buf);
const u8  = new Uint8Array(buf);       // same bytes, different view
u32[0] = 0xDEADBEEF;
u8[0];                                 // 0xEF on little-endian systems
```

`SharedArrayBuffer` backs a buffer that can be shared between threads (Worker / Worker Threads) — **real shared memory**, not message-passed copies. You need `Atomics` for races:

```js
// In worker threads — see the `worker_threads` module
const shared = new SharedArrayBuffer(4);
const counter = new Int32Array(shared);
Atomics.add(counter, 0, 1);            // atomic increment across threads
Atomics.wait(counter, 0, 0);           // sleep until counter[0] !== 0
Atomics.notify(counter, 0, 1);         // wake one waiter
```

Use cases: lock-free ring buffers between workers, high-throughput parsing pipelines, signaling. The browser restricts `SharedArrayBuffer` to cross-origin-isolated contexts (COOP/COEP headers) due to Spectre — Node has no such restriction.

---

