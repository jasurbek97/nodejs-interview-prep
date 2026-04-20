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

