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

