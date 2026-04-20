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

