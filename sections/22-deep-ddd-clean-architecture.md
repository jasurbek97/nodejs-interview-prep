## 22. Deep Dive — DDD & Clean Architecture

### Q196. Why DDD? Strategic vs tactical

**Strategic DDD** — splits a complex problem domain into **bounded contexts**, each with its own model and ubiquitous language. The point: prevent one giant model that means different things to different teams (`Customer` in Sales ≠ `Customer` in Billing).

**Tactical DDD** — within one bounded context, the building blocks: **entities, value objects, aggregates, domain events, repositories, services**. The point: a domain model that's expressive, behavior-rich, and decoupled from infrastructure.

You don't need both. A CRUD admin tool needs neither. A complex business domain (insurance, logistics, fintech) usually needs strategic DDD even if tactical patterns are kept light.

### Q197. Bounded contexts and the context map

A **bounded context** is the explicit boundary inside which a model is consistent and unambiguous. Often (not always) maps to a microservice.

A **context map** documents the relationships between contexts:
- **Partnership** — two teams succeed or fail together; coordinate releases.
- **Customer/Supplier** — downstream depends on upstream; upstream takes downstream's needs into account.
- **Conformist** — downstream uses upstream's model as-is (no leverage).
- **Anti-corruption Layer (ACL)** — downstream wraps upstream behind an adapter so the upstream's model doesn't pollute the local one.
- **Shared Kernel** — two contexts share a small piece of model. Risky — needs strict change discipline.
- **Open Host Service** — upstream exposes a published, versioned API for many downstreams.
- **Published Language** — a shared, formal contract (events, OpenAPI) between contexts.

Context boundaries usually align with **subdomains**: **core** (your competitive advantage — invest), **supporting** (necessary but generic — build but light), **generic** (commodities — buy/use SaaS).

### Q198. Entities vs value objects

- **Entity** — has identity that persists across changes. `Order(id=42)` is the same order whether the status is `pending` or `shipped`. Compared by ID.
- **Value object** — defined entirely by its attributes. `Money(100, "USD")` is the same as any other `Money(100, "USD")`. Compared by value, immutable, no identity.

```ts
class Money {
  constructor(readonly amount: number, readonly currency: string) {
    if (amount < 0) throw new InvalidMoney('negative');
    Object.freeze(this);
  }
  add(other: Money): Money {
    if (other.currency !== this.currency) throw new CurrencyMismatch();
    return new Money(this.amount + other.amount, this.currency);
  }
}
```

Push behavior into value objects (`money.add()`, `email.normalize()`) — they're cheap, safe, testable.

### Q199. Aggregates and aggregate roots — the rules

An **aggregate** is a cluster of entities and value objects treated as a single transactional unit. It has one **root entity**; outside callers can only reference the root, never inner objects.

The four invariant rules:
1. **One aggregate per transaction.** If you must change two aggregates, it implies eventual consistency (domain event → handler updates the second).
2. **Reference other aggregates by ID, not by object reference.** `Order` holds `customerId`, not a `Customer` instance.
3. **The root enforces invariants.** All mutations go through root methods; inner state is encapsulated.
4. **Keep aggregates small.** Big aggregates lock-contend and load slowly. If invariants don't span two entities, they're separate aggregates.

```ts
class Order {  // aggregate root
  private items: OrderItem[] = [];
  private constructor(readonly id: OrderId, readonly customerId: CustomerId) {}

  addItem(productId: ProductId, qty: number, price: Money) {
    if (this.items.length >= 100) throw new TooManyItems();
    this.items.push(new OrderItem(productId, qty, price));
    this.addEvent(new ItemAddedToOrder(this.id, productId, qty));
  }
}
```

### Q200. Domain events

Aggregates record events when something domain-relevant happens. Events:
- Are **named in past tense** (`OrderPlaced`, `PaymentCaptured`).
- Carry the data subscribers need (immutable, serializable).
- Are dispatched **after the transaction commits** (otherwise a rollback leaves a phantom event).

Pattern: aggregate.addEvent() → repository collects events on save → after COMMIT, publish to in-process bus + outbox table for cross-service.

This is where "single aggregate per transaction" stops feeling like a constraint — events make multi-aggregate workflows natural.

### Q201. Repositories — persistence ignorance

A repository looks like an in-memory collection of aggregates: `orderRepo.findById(id)`, `orderRepo.save(order)`. It hides whether the data lives in Postgres, DynamoDB, or memory.

```ts
interface OrderRepository {
  findById(id: OrderId): Promise<Order | null>;
  save(order: Order): Promise<void>;
}
```

Implementation lives in the **infrastructure** layer (e.g., `PostgresOrderRepository`). Domain code depends only on the interface. This makes it possible to test domain logic without a database and swap persistence later.

Don't expose query builders or `findBy<RandomCombination>` — those are read-side concerns. Use a separate **query service** / read model for those (CQRS).

### Q202. Application services vs domain services

- **Application service** (use case) — orchestrates a single transaction: load aggregates, call domain methods, save, dispatch events. Thin, no business rules.
- **Domain service** — business logic that doesn't naturally belong to an entity (`PriceCalculator`, `TransferFundsService` operating on two `Account` aggregates). Stateless, in the domain layer.

```ts
class PlaceOrderUseCase {
  constructor(private orders: OrderRepository, private events: EventBus) {}
  async execute(cmd: PlaceOrderCommand) {
    const order = Order.place(cmd.customerId, cmd.items);  // domain logic in entity
    await this.orders.save(order);
    await this.events.publishAll(order.pullEvents());
  }
}
```

### Q203. Clean Architecture — the dependency rule

Layers, with dependencies pointing **inward only**:

```
┌─────────────────────────────────────────────────────┐
│  Frameworks & Drivers (web, db, devices, UI)        │
│  ┌───────────────────────────────────────────────┐  │
│  │  Interface Adapters (controllers, presenters, │  │
│  │   gateways, ORM mappings)                     │  │
│  │  ┌───────────────────────────────────────┐    │  │
│  │  │  Application (use cases)              │    │  │
│  │  │  ┌──────────────────────────────┐     │    │  │
│  │  │  │  Domain (entities, VOs,      │     │    │  │
│  │  │  │   aggregates, domain events) │     │    │  │
│  │  │  └──────────────────────────────┘     │    │  │
│  │  └───────────────────────────────────────┘    │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

Domain has zero imports from outer layers. Use cases depend on domain. Adapters implement interfaces declared in inner layers (dependency inversion). This is what makes the inside testable in isolation and replaceable from the outside.

### Q204. Hexagonal (ports & adapters)

Same idea, different vocabulary. The core is the **application + domain**. Around it:

- **Driving (primary) ports** — interfaces the application exposes (use case interfaces). **Driving adapters**: HTTP controllers, CLI, message consumers — they invoke the port.
- **Driven (secondary) ports** — interfaces the application needs (`OrderRepository`, `PaymentGateway`). **Driven adapters**: Postgres repo, Stripe client, in-memory fake.

The application sits between, agnostic of either side. You can run it against an HTTP adapter in prod and a test runner adapter in CI, against Postgres in prod and an in-memory adapter in tests — same core.

### Q205. NestJS folder layout for DDD

```
src/
  modules/
    orders/
      domain/                  # pure TS, no Nest decorators
        order.aggregate.ts
        order-item.vo.ts
        money.vo.ts
        events/
          order-placed.event.ts
        order.repository.ts    # interface
      application/
        commands/
          place-order/
            place-order.command.ts
            place-order.handler.ts
        queries/
          get-order/
            get-order.query.ts
            get-order.handler.ts
        ports/
          payment-gateway.port.ts
      infrastructure/
        persistence/
          order.entity.ts      # ORM/typeorm/prisma model
          order.repository.impl.ts
          order.mapper.ts      # ORM model ↔ aggregate
        gateways/
          stripe-payment.gateway.ts
      presentation/
        http/
          orders.controller.ts
          dto/
        events/
          order-placed.consumer.ts
      orders.module.ts
  shared/
    kernel/                    # cross-cutting domain primitives (Money, Email, …)
```

`domain/` has zero `@Injectable()` — it's pure. Wiring lives in `orders.module.ts` (`provide: ORDER_REPOSITORY, useClass: PostgresOrderRepository`). Use cases get the interface, not the implementation.

### Q206. CQRS — when and how

**Command-Query Responsibility Segregation** — separate write model (rich domain, transactional) from read model (denormalized, optimized for queries).

```
Write side:  Command ─► Use case ─► Aggregate ─► Repository ─► DB
                                       │
                                       └─► Domain event ─► Projector
                                                              ↓
Read side:   Query ─────────────────────────────────► Read model (cache, ES, denormalized table)
```

When CQRS pays off:
- Read patterns are very different from the write model (e.g., dashboards joining 8 tables).
- Read load >> write load.
- You're already eventing for other reasons.

When it's overkill: simple CRUD with the same model on both sides.

NestJS has `@nestjs/cqrs` if you want bus + handler scaffolding, but you can roll it yourself in 30 lines.

### Q207. Event sourcing — when to reach for it

Instead of storing the current state, store the sequence of events. Current state = fold(events).

```
Event stream for Order(42):
  OrderPlaced { items: [...] }
  ItemAdded { product: X }
  PaymentCaptured { amount: ... }
  OrderShipped { tracking: ... }
```

**Pros:** perfect audit log, time travel, easy to add new projections retroactively, natural fit with CQRS and DDD events.

**Cons:** schema evolution is harder (you can never delete old event shapes — only "upcast" them), querying current state requires projections, eventual consistency between write store and read models, snapshots needed for long streams.

Use it for domains where the **history is the truth**: ledgers, audit-heavy systems, regulated domains. Avoid it for "we wanted CRUD but with extra steps."

### Q208. Anti-corruption layer

When integrating with a legacy or external system whose model conflicts with yours, build an ACL — adapters + mapping that translate the external model into your bounded context's model on the way in and back on the way out.

```
External "Customer" model                Your "Buyer" aggregate
  { firstName, lastName,                   Buyer {
    addr1, addr2, addr3,         ─►          name: PersonName,
    flagsBitmask, ... }                      address: Address,
                                             tier: BuyerTier }
```

The ACL lives in the **infrastructure** layer. Without it, the external model leaks into your domain and corrupts it (hence the name).

### Q209. Anemic vs rich domain model

**Anemic** — entities are property bags; logic lives in service classes (`OrderService.addItem(order, item)`). Looks like Java circa 2005. Easy to start with, terrible to maintain — invariants get violated because there's no single guardian.

**Rich** — behavior lives on the entity (`order.addItem(item)`). The entity enforces its invariants. This is what DDD asks for.

Sign of an anemic model: lots of `getOrder…(): X; setOrder…(x): void` and a sibling `Service` doing all the logic. Sign of a rich model: aggregates expose verbs, not setters.

### Q210. Transaction boundaries in DDD

The application service starts and commits the transaction. **One aggregate per transaction**. Multi-aggregate workflows go through domain events + a separate handler that opens its own transaction:

```
PlaceOrderUseCase tx:
  - load Order
  - order.place()
  - save Order
  - publish events  (after commit, via outbox)

InventoryReservationOnOrderPlaced handler tx:
  - load InventoryItem
  - inventory.reserve()
  - save InventoryItem
```

If the second tx fails, you compensate (cancel the order, refund payment) — not roll back the first. This is the saga pattern in DDD clothing.

---

