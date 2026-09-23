# Structured Logging

## Core Idea

`console.log` is fine for local development, but in production logs need to be searched, filtered, and aggregated — which requires a structured (JSON) format, severity levels, and an end-to-end request identifier.

## Structured Logging via Pino

`nestjs-pino` replaces Nest's built-in logger with a structured one, at minimal performance overhead:

```ts
@Module({
  imports: [
    LoggerModule.forRoot({
      pinojsonHttp: { level: process.env.LOG_LEVEL || 'info' },
    }),
  ],
})
export class AppModule {}
```

```ts
@Injectable()
export class OrderService {
  constructor(private readonly logger: Logger) {}

  async createOrder(dto: CreateOrderDto) {
    this.logger.log({ orderId: dto.id, amount: dto.amount }, 'Order created');
    // JSON output: { "level": "info", "orderId": "...", "amount": ..., "msg": "Order created" }
  }
}
```

A structured log (instead of a plain text line) lets you search and filter by fields (`orderId`, `userId`) in a log aggregation system, instead of parsing text with regular expressions.

## Correlation ID (Request ID)

A single HTTP request often passes through several services or internal calls. Without an end-to-end identifier, it's impossible to collect all the logs related to one request in one place.

```ts
// Middleware that generates/propagates a request ID
@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    req['requestId'] = req.headers['x-request-id'] ?? randomUUID();
    next();
  }
}
```

This identifier is then passed to every log entry within the request — most conveniently via `AsyncLocalStorage` (e.g., `nestjs-cls`), so you don't have to thread it manually through every function call:

```ts
this.logger.log({ requestId: this.cls.get('requestId'), orderId: order.id }, 'Order created');
```

## Log Levels

- **error** — something broke and needs attention (an exception, an unavailable dependency).
- **warn** — a potential problem, but the application keeps running (a deprecated API, a slow query).
- **info** — significant business events (order created, user registered).
- **debug** — diagnostic detail, disabled in production by default.

Logging "everything" at the `info` level is just as harmful as not logging enough — the important signal gets lost in the noise.

## What Must NEVER Be Logged

Passwords, access tokens, card numbers, users' personal data — the same things that must never be returned in a response (`references/security/security-hardening.md`). A mistake here is a typical path for sensitive data leaks, since logs are often kept longer and have broader access than the database itself.

```ts
// ❌ Unsafe: the entire DTO is logged "as is", including the password
this.logger.log({ dto }, 'Registration attempt');
```

```ts
// ✅ Safe: only what's actually needed for diagnostics is logged
this.logger.log({ email: dto.email }, 'Registration attempt');
```

## Why This Matters

Centralized, structured logging is what makes `references/operations/health-monitoring.md` (Health Checks, Prometheus/Grafana) actually usable for diagnostics: metrics show WHAT broke, and logs with a correlation ID show WHY.

<!-- Open for expansion: log aggregation (ELK/Loki), integration with distributed tracing (OpenTelemetry) -->
