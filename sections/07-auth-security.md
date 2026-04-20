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

