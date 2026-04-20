## 5. NestJS

### Q43. What is NestJS and why use it?

A progressive Node framework written in TypeScript, inspired by Angular. It provides:
- **Dependency Injection** out of the box
- **Modular architecture** (everything is a module)
- **Decorators** for routes, validation, docs
- Built-in support for **GraphQL, WebSockets, microservices, CQRS**
- Testable by design

### Q44. Core building blocks

- **Module** — group of related code (`@Module`).
- **Controller** — HTTP route handler (`@Controller`).
- **Provider / Service** — business logic, injectable (`@Injectable`).
- **Pipe** — transform/validate input (`@UsePipes`).
- **Guard** — authorization (`@UseGuards`).
- **Interceptor** — wrap execution (logging, caching, transform response).
- **Filter** — exception handling (`@UseFilters`).

### Q45. Request lifecycle in NestJS

```
Middleware → Guards → Interceptors (before) → Pipes → Controller → Service
                           ↓
                   Interceptors (after) → Filters (if error) → Response
```

### Q46. How does DI work in NestJS?

Based on **constructor injection**. Under the hood it reads TypeScript's emitted metadata (`emitDecoratorMetadata`) to know which tokens to inject.

```ts
@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User) private repo: Repository<User>,
    private logger: Logger,
  ) {}
}
```

Scopes: `DEFAULT` (singleton), `REQUEST`, `TRANSIENT`.

### Q47. Custom decorator example

```ts
export const CurrentUser = createParamDecorator(
  (data: unknown, ctx: ExecutionContext) =>
    ctx.switchToHttp().getRequest().user,
);

@Get('me')
profile(@CurrentUser() user: User) { return user; }
```

### Q48. Guards vs Middleware vs Interceptors

| Concern                   | Use             |
| ------------------------- | --------------- |
| Generic request processing (logging, parsing) | Middleware |
| Authorization decisions ("can this run?")     | Guards     |
| Cross-cutting (timing, caching, transform)    | Interceptors |
| Transforming/validating input                 | Pipes      |
| Handling thrown exceptions                    | Filters    |

### Q49. How would you test a NestJS service?

```ts
const module = await Test.createTestingModule({
  providers: [
    UsersService,
    { provide: getRepositoryToken(User), useValue: mockRepo },
  ],
}).compile();

const service = module.get(UsersService);
```

---

