## 9. ORMs — Prisma, Sequelize, TypeORM

### Q81. Compare the three

| Feature              | Prisma                   | Sequelize               | TypeORM                |
| -------------------- | ------------------------ | ----------------------- | ---------------------- |
| Type safety          | Excellent (generated)    | Weak                    | Good (decorators)      |
| Schema source        | `schema.prisma` DSL      | JS models               | Decorated classes      |
| Migrations           | First-class              | CLI                     | CLI                    |
| Active Record / DM   | Data Mapper              | Active Record           | Both                   |
| Raw SQL escape       | `$queryRaw`              | `sequelize.query`       | `query`                |
| Community            | Fastest growing          | Oldest, huge            | Stable, NestJS default |
| Downsides            | Extra generate step, opinionated | Type-unsafe, clunky | Maintenance concerns |

### Q82. Prisma example

```prisma
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  posts     Post[]
  createdAt DateTime @default(now())
}
```

```ts
const user = await prisma.user.create({
  data: { email: 'a@b.com', posts: { create: [{ title: 'Hi' }] } },
  include: { posts: true },
});
```

### Q83. Migrations — best practices

- **Never** edit a migration that's been applied in production. Create a new one.
- Forward-compatible deploys: **expand → migrate → contract**.
  1. Add new column nullable.
  2. Dual-write.
  3. Backfill.
  4. Switch reads.
  5. Drop old column in a later deploy.
- Keep migrations small and fast — avoid long table locks.

### Q84. Transactions with an ORM

Prisma:
```ts
await prisma.$transaction(async (tx) => {
  await tx.account.update({ where: { id: from }, data: { balance: { decrement: amount } } });
  await tx.account.update({ where: { id: to   }, data: { balance: { increment: amount } } });
  if ((await tx.account.findUnique({ where: { id: from } }))!.balance < 0) throw new Error('overdraft');
});
```

TypeORM has `queryRunner.startTransaction()` / `commitTransaction()` and `dataSource.transaction(cb)`.

### Q85. When would you drop the ORM and write raw SQL?

- Reports with complex joins, window functions, CTEs.
- Bulk operations (`INSERT ... ON CONFLICT`, `COPY`).
- Queries where the ORM generates inefficient SQL (check `EXPLAIN`).
- DB-specific features (PostgreSQL ranges, JSONB paths, `LATERAL` joins).

---

