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

