# Request Scoping vs Singleton

## The essence of the approach

NestJS manages the DI container through provider **scope** (lifetime). Understanding this mechanism is critical for application performance.

- **Singleton (default).** The provider is created once for the entire lifetime of the application and reused for all requests. Most services are like this.
- **Request scope.** A new instance of the provider is created for every incoming request.
- **Transient scope.** A new instance is created every time the provider is injected (even within a single request).

```ts
// Singleton — the default, no extra configuration needed
@Injectable()
export class UsersService {}
```

```ts
// Request-scoped — a new instance for every HTTP request
@Injectable({ scope: Scope.REQUEST })
export class RequestContextService {
  userId: string; // safe to hold state that belongs to a specific request
}
```

## Why this matters for performance

`REQUEST` scope significantly affects performance: the provider (and the entire chain of classes that consume it) is recreated on every request, instead of existing just once. Under high load, this is a noticeable CPU/memory cost compared to the default Singleton approach.

## When REQUEST scope is actually needed

Use `REQUEST` scope only when it's truly necessary — for example, to log context for a specific user (user ID, request ID, tenant ID in a multi-tenant application) throughout the request:

```ts
@Injectable({ scope: Scope.REQUEST })
export class LoggerService {
  constructor(@Inject(REQUEST) private readonly request: Request) {}

  log(message: string) {
    console.log(`[user:${this.request['userId']}] ${message}`);
  }
}
```

In all other cases, stick with Singleton — it's the default for a reason, and it should be changed deliberately, not "just in case."

<!-- Open for expansion: how REQUEST scope "leaks" into every provider that consumes it, the alternative via AsyncLocalStorage (nestjs-cls) without REQUEST scope -->
