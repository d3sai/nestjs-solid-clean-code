# Dependency Injection anti-patterns

Dependency Injection is a powerful pattern, but using it incorrectly damages an application's architecture no less than not using it at all. The main anti-patterns are:

## 1. Service Locator (hidden dependency)

Instead of injecting concrete dependencies through the constructor, the entire `ModuleRef`/`Injector` is passed into the class, and the needed service is "fetched" manually at runtime.

```ts
// ❌ Bad: the class's dependencies are implicit, invisible from the constructor
@Injectable()
export class OrderService {
  constructor(private readonly moduleRef: ModuleRef) {}

  async createOrder(dto: CreateOrderDto) {
    const paymentService = this.moduleRef.get(PaymentService, { strict: false });
    return paymentService.charge(dto.amount);
  }
}
```

```ts
// ✅ Good: an explicit dependency via the constructor
@Injectable()
export class OrderService {
  constructor(private readonly paymentService: PaymentService) {}

  async createOrder(dto: CreateOrderDto) {
    return this.paymentService.charge(dto.amount);
  }
}
```

Consequence of this anti-pattern: it's impossible to tell what the class depends on without reading the entire implementation of its methods, and swapping a dependency in a test turns into a fight with `ModuleRef` instead of a simple `useValue`.

## 2. Circular Dependencies

Service A depends on Service B, and Service B depends on Service A. In NestJS, this often leads to initialization errors (one of the dependencies ends up `undefined`).

```ts
// ❌ UsersService and OrdersService depend directly on each other
@Injectable()
export class UsersService {
  constructor(private readonly ordersService: OrdersService) {}
}

@Injectable()
export class OrdersService {
  constructor(private readonly usersService: UsersService) {}
}
```

A quick fix is `forwardRef()`:

```ts
@Injectable()
export class UsersService {
  constructor(
    @Inject(forwardRef(() => OrdersService))
    private readonly ordersService: OrdersService,
  ) {}
}
```

But `forwardRef()` only removes the symptom. The correct fix is to revisit the design: extract the shared logic into a third service that both depend on, or replace the direct call with an event (see `references/architecture/event-driven-architecture.md`) to break the cycle entirely.

## 3. DI Container Abuse (overloading the container)

DI is used to pass configuration, constants, or arbitrary objects that aren't full-fledged services.

```ts
// ❌ Bad: the container turns into a "junk drawer" of arbitrary values
@Module({
  providers: [
    { provide: 'API_KEY', useValue: 'sk_live_...' },
    { provide: 'MAX_RETRIES', useValue: 3 },
    { provide: 'FEATURE_FLAG_NEW_CHECKOUT', useValue: true },
  ],
})
export class AppModule {}
```

There's a specialized tool for settings — `ConfigService` (see `references/architecture/configuration.md`) — rather than injecting a separate provider for each value:

```ts
// ✅ Good: all settings go through a single entry point
@Injectable()
export class PaymentService {
  constructor(private readonly configService: ConfigService) {}

  private get apiKey() {
    return this.configService.get<string>('API_KEY');
  }
}
```

## 4. Constructor Bloat (too many dependencies)

If a constructor takes 10+ services, that's a sign the Single Responsibility Principle is being violated (see `references/solid/srp.md`). The class is doing too much.

```ts
// ❌ Bad: the class is clearly responsible for far too many different things
@Injectable()
export class OrderService {
  constructor(
    private readonly orderRepository: OrderRepository,
    private readonly paymentService: PaymentService,
    private readonly emailService: EmailService,
    private readonly smsService: SmsService,
    private readonly analyticsService: AnalyticsService,
    private readonly inventoryService: InventoryService,
    private readonly loyaltyService: LoyaltyService,
    private readonly invoiceService: InvoiceService,
    private readonly auditLogService: AuditLogService,
    private readonly notificationService: NotificationService,
  ) {}
}
```

The solution is to split the class into smaller ones, each with its own narrow responsibility (for example, extracting notification orchestration into a separate `OrderNotificationService`, which itself depends on `EmailService`/`SmsService`/`NotificationService`), rather than keeping all 10 dependencies in one place.

## 5. Injection of Fat Objects

Injecting an entire module or a bulky object instead of a narrowly focused service. A dependency should be as abstract and lightweight as possible — this is the same principle as ISP (see `references/solid/isp.md`), just applied not to an interface but to what is actually injected.

```ts
// ❌ Bad: the class depends on a "fat" object but uses only one field from it
@Injectable()
export class InvoiceService {
  constructor(private readonly appConfig: AppConfig) {} // the entire application config

  generate() {
    const currency = this.appConfig.billing.invoice.defaultCurrency;
    // ...
  }
}
```

```ts
// ✅ Good: the dependency is narrow and focused on what's actually needed
@Injectable()
export class InvoiceService {
  constructor(private readonly configService: ConfigService) {}

  generate() {
    const currency = this.configService.get<string>('DEFAULT_CURRENCY');
  }
}
```

## Why all of these anti-patterns are harmful

- **Difficult testing.** Hidden or "fat" dependencies can't easily be swapped for a mock — you have to mock entire objects or the whole `ModuleRef` instead of one narrow interface.
- **High coupling.** Components become inseparably linked, which hinders their reuse and independent change.
- **Poor readability.** It's impossible to tell what a class actually needs to function just by looking at the constructor — and that's the main benefit of DI, which these anti-patterns destroy.

<!-- Open for expansion: how a linter (eslint-plugin) can automatically catch some of these anti-patterns, a step-by-step example of refactoring Constructor Bloat -->
