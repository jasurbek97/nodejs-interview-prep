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

