# Guards, Pipes, and Interceptors

NestJS's architecture is built around these three tools, which let you separate cross-cutting logic (authorization, validation, response transformation) from the business logic of controllers and services — a direct application of SRP at the whole-application level.

## 1. Guards

Guards are responsible for authentication and authorization. They decide whether a request should be allowed through to the handler (controller handler), based on some condition (e.g., the user's role or the presence of a token).

- **When they run:** first in the request chain, right after Middleware.
- **Key function:** `canActivate()` returns `boolean` or `Promise<boolean>`. If `false` — the request is rejected (a `ForbiddenException` is thrown).

```ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRole = this.reflector.get<string>('role', context.getHandler());
    const request = context.switchToHttp().getRequest();
    return request.user?.role === requiredRole;
  }
}
```

```ts
@UseGuards(RolesGuard)
@Get('admin')
adminOnly() {
  // runs only if canActivate() returned true
}
```

## 2. Pipes

Pipes are used for two purposes:

1. **Data transformation** — converting input data to the required format (e.g., the string `"1"` → the number `1`).
2. **Validation** — checking that input data matches the expected schema (usually via `class-validator`, see `references/data-validation/dto-validation.md`).

- **When they run:** before the handler method is called.
- **Key function:** `transform()`.

```ts
@Injectable()
export class ParseIntPipe implements PipeTransform {
  transform(value: string): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed: not a number');
    }
    return val;
  }
}
```

```ts
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  // id arrives here already guaranteed to be a number
}
```

## 3. Interceptors

The most powerful of the three tools — it has access to the request both before and after the handler method runs. It can modify the request's result, handle errors, or add logging logic.

- **When they run:** they wrap the execution of the handler method (i.e., they have "before" code and "after" code).
- **Key capabilities:**
  - Intercepting and modifying the response (e.g., wrapping the result in a standard `{ data, meta }` structure).
  - Handling exceptions (you can catch an error and return your own response).
  - Measuring request execution time.

```ts
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const start = Date.now();
    const request = context.switchToHttp().getRequest();

    return next.handle().pipe(
      tap(() => {
        console.log(`${request.method} ${request.url} — ${Date.now() - start}ms`);
      }),
    );
  }
}
```

```ts
@Injectable()
export class TransformResponseInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map((data) => ({ data, timestamp: new Date().toISOString() })),
    );
  }
}
```

`next.handle()` is the call to the handler method itself; everything written before it in `intercept()` runs "before," and the logic inside `.pipe(...)` runs "after" the handler completes.

## Quick Comparison

| Tool | Question it answers |
|---|---|
| **Guards** | "Do you have access?" |
| **Pipes** | "Is the input data correct?" |
| **Interceptors** | "What to do before and after execution, and whether to modify the result?" |

## Execution Order in the Request Lifecycle

```
Request → Middleware → Guards → Interceptors (before) → Pipes → Controller → Service
      → Interceptors (after) → (Exception Filter, if an exception occurred) → Response
```

Guards run before Pipes and Interceptors for a reason: there's no point validating or transforming the data of a request the user doesn't even have access to. The `Exception Filter` (see `references/cross-cutting/exception-filters.md`) stands apart from this linear flow — it catches an exception thrown at any of the preceding stages.

<!-- Open for expansion: global vs. local Guards/Pipes/Interceptors, custom parameter decorators (createParamDecorator), an example of ValidationPipe combined with whitelist -->
