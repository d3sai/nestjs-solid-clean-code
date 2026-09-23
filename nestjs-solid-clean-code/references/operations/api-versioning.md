---
note: Section added beyond the originally agreed-upon scope — enterprise/senior-level production readiness, at the user's request.
---

# API Versioning

## Core Idea

NestJS's built-in API versioning mechanism (`/v1/`, `/v2/`) allows you to roll out breaking changes without stopping existing client applications that still use the previous version.

## Example in NestJS

```ts
// main.ts
app.enableVersioning({
  type: VersioningType.URI,
  defaultVersion: '1',
});
```

```ts
@Controller({ path: 'orders', version: '1' })
export class OrdersV1Controller {
  @Get()
  findAll() {
    // the old response shape, already used by existing clients
  }
}

@Controller({ path: 'orders', version: '2' })
export class OrdersV2Controller {
  @Get()
  findAll() {
    // the new, incompatible response shape
  }
}
```

Both controllers run at the same time: old clients keep calling `/v1/orders`, new ones call `/v2/orders`, and neither breaks during the transition.

## Why This Matters

Without versioning, any change to the API contract (renaming a field, changing the response structure) forces every client to be updated at the same time as the backend release — which in practice is rarely feasible synchronously (mobile apps, third-party integrations). Versioning buys time for a gradual migration.

<!-- Open for expansion: versioning via header/media-type instead of URI, a deprecation strategy for old versions -->
