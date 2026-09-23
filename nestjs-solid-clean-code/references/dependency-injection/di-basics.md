# Dependency Injection in NestJS: the basics

## The essence of the mechanism

Dependency Injection (DI) is a mechanism built into NestJS for managing dependencies between classes. Instead of a class creating its own dependencies (`new SomeService()`), it simply declares what it needs, and the framework creates the required instance and passes it in.

## Three steps to implement DI

### 1. Creating a service

A class is marked with the `@Injectable()` decorator — this makes it a "provider" that Nest can create and inject into other classes.

```ts
@Injectable()
export class PaymentService {
  process() {
    /* payment processing logic */
  }
}
```

### 2. Registering it in a module

The class is added to the `providers` array of the relevant module, so that Nest knows it exists and can create an instance for it (Nest builds its dependency graph based specifically on this array).

```ts
@Module({
  providers: [PaymentService],
})
export class OrderModule {}
```

### 3. Injecting it through the constructor

The needed service is simply specified in the constructor of a controller or another service — Nest automatically creates the instance and supplies it.

```ts
@Controller('orders')
export class OrderController {
  constructor(private readonly paymentService: PaymentService) {}
}
```

## Why this matters

- **Testability.** A real service can easily be swapped for a "mock" during testing — there's no need to change the consuming class itself, just swap the provider in the test module.
- **Loose coupling.** Classes don't create their own dependencies via `new`. This directly implements the **Dependency Inversion Principle (DIP)**: components depend on abstractions, not on concrete implementations (see `references/solid/dip.md`).

## DI via tokens: not just classes, but abstractions too

DI in Nest isn't limited to concrete classes — you can also inject interfaces or abstract classes using tokens and the `@Inject()` decorator. This makes the system flexible for swapping strategies "on the fly" — for example, replacing one payment gateway with another without changing the code that uses it.

```ts
export const PAYMENT_GATEWAY = Symbol('PAYMENT_GATEWAY');

export interface PaymentGateway {
  charge(amount: number): Promise<void>;
}

@Injectable()
export class StripeGateway implements PaymentGateway {
  async charge(amount: number) { /* ... */ }
}

@Module({
  providers: [
    { provide: PAYMENT_GATEWAY, useClass: StripeGateway },
  ],
})
export class PaymentModule {}
```

```ts
@Injectable()
export class OrderService {
  constructor(
    @Inject(PAYMENT_GATEWAY) private readonly gateway: PaymentGateway,
  ) {}
}
```

Replacing `StripeGateway` with a different payment gateway happens by changing a single line (`useClass`) in the module — without any changes to `OrderService`. For more on common mistakes that break DI (for example, `new Service()` instead of injection), see `references/dependency-injection/di-anti-patterns.md`.

<!-- Open for expansion: useFactory, useValue, scope (REQUEST/TRANSIENT), circular dependencies via forwardRef -->
