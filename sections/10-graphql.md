## 10. GraphQL

### Q86. What is GraphQL and how is it different from REST?

A query language and runtime where clients specify **exactly** which fields they need. One endpoint, strongly-typed schema.

| Aspect        | REST                       | GraphQL                             |
| ------------- | -------------------------- | ----------------------------------- |
| Endpoints     | Many                       | One                                 |
| Over/underfetching | Common                | Minimal                             |
| Versioning    | Explicit (`/v1/`)          | Evolve schema (deprecate fields)    |
| Caching       | HTTP caching out-of-box    | Harder; needs extra infra           |
| Error codes   | HTTP statuses              | Always 200; errors in body          |
| File uploads  | Trivial                    | Requires extension (`graphql-upload`) |

### Q87. Schema, queries, mutations, subscriptions

```graphql
type User  { id: ID!, email: String!, posts: [Post!]! }
type Post  { id: ID!, title: String!, author: User! }

type Query { user(id: ID!): User }
type Mutation { createPost(input: CreatePostInput!): Post! }
type Subscription { postAdded: Post! }
```

### Q88. Resolvers and the N+1 problem

Each field can have a resolver. If a query returns 100 posts, the naive `post.author` resolver runs 100 SQL queries — the GraphQL N+1 problem.

**Fix: DataLoader.** It batches and caches (per-request).

```ts
const userLoader = new DataLoader(async (ids) => {
  const users = await db.user.findMany({ where: { id: { in: ids as string[] } } });
  const map = new Map(users.map(u => [u.id, u]));
  return ids.map(id => map.get(id as string));
});

// resolver
author: (post, _, ctx) => ctx.loaders.user.load(post.authorId),
```

### Q89. Apollo Server setup (Node)

```ts
const server = new ApolloServer({ typeDefs, resolvers });
const { url } = await startStandaloneServer(server, {
  context: async ({ req }) => ({
    user: await getUserFromToken(req.headers.authorization),
    loaders: makeLoaders(),
  }),
});
```

### Q90. Authorization in GraphQL

- **Field-level** — check in resolvers (or directives like `@auth(role: ADMIN)`).
- **Schema-level** — gate entire types.
- Keep **business rules** in services, not resolvers.
- Avoid exposing internal IDs/raw DB shapes — use DTOs.

### Q91. Schema stitching vs Federation

- **Stitching** — merge remote schemas into a gateway. Older, simpler, less scalable.
- **Federation (Apollo)** — each subgraph owns its types; gateway composes them. Better for microservices.

### Q92. GraphQL downsides / gotchas

- Query complexity → potential DoS. Use depth limiting, cost analysis, persisted queries.
- HTTP caching is hard (queries are POSTs with varied bodies).
- Client must know GraphQL; learning curve.
- Introspection can leak schema — disable in prod if unused.

---

