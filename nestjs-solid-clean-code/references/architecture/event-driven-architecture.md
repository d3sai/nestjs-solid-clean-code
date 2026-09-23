---
note: Section added beyond the originally agreed table of contents — Enterprise/Senior-level architecture, added at the user's request.
---

# Event-Driven Architecture (EDA)

## Core Idea

Instead of one service directly calling another (for example, `OrderService` directly calling `EmailService`), a service publishes an event (for example, `OrderCreatedEvent`). Other modules (`Email`, `Analytics`) subscribe to that event and react independently of one another.

This makes the system decoupled (loosely coupled) — the module publishing the event knows nothing about who is listening to it or how many listeners there are.

## Example in NestJS (`@nestjs/event-emitter`)

The implementation consists of three steps:

### 1. Registering the Module

`EventEmitterModule` is wired up once, in the application's root module:

```ts
@Module({
  imports: [EventEmitterModule.forRoot()],
})
export class AppModule {}
```

### 2. Publishing and Subscribing to an Event

```ts
// ❌ Direct coupling: OrderService knows about all of its "consumers"
@Injectable()
export class OrderService {
  constructor(
    private readonly emailService: EmailService,
    private readonly analyticsService: AnalyticsService,
  ) {}

  async createOrder(dto: CreateOrderDto) {
    const order = await this.orderRepository.save(dto);
    await this.emailService.sendOrderConfirmation(order);
    await this.analyticsService.trackOrderCreated(order);
    return order;
  }
}
```

```ts
// ✅ Event-driven: OrderService only announces that something happened
@Injectable()
export class OrderService {
  constructor(
    private readonly orderRepository: OrderRepository,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async createOrder(dto: CreateOrderDto) {
    const order = await this.orderRepository.save(dto);
    // Publishing the event — EventEmitter2 after the operation completes
    this.eventEmitter.emit('order.created', new OrderCreatedEvent(order));
    return order;
  }
}
```

### 3. Subscribing to the Event

The `@OnEvent` decorator automatically calls the handler method when the event is emitted — subscribing requires no manual registration, just the decorator on the provider's method.

```ts
@Injectable()
export class EmailListener {
  @OnEvent('order.created')
  async handleOrderCreated(event: OrderCreatedEvent) {
    // sending the email — the Email module decides on its own what to do with the event
  }
}

@Injectable()
export class AnalyticsListener {
  @OnEvent('order.created')
  async handleOrderCreated(event: OrderCreatedEvent) {
    // tracking the metric — independently of EmailListener
  }
}
```

## Why This Matters

- **Loose coupling.** `OrderService` can be changed, tested, and deployed independently of how many modules listen to the `order.created` event. Adding a new listener (for example, `LoyaltyPointsListener`) requires no change whatsoever in `OrderService`.
- **Extensibility (OCP).** A new reaction to the event is a new listener, not a modification to an existing service. See `references/solid/ocp.md`.
- **Module boundaries.** This is one of the tools for communication between Bounded Contexts without direct dependencies — see `references/architecture/module-boundaries.md`.

## Trade-off

The event-driven approach makes it harder to trace the flow of execution: to understand what happens after `order.created`, you need to find all the listeners scattered across the codebase — unlike a direct method call, where the execution path is obvious right away. For simple, small applications, a direct service call is often more readable and sufficient.

<!-- Open for expansion: async vs sync events, error handling in listeners, an event bus between microservices (Kafka/RabbitMQ) vs the internal EventEmitter -->
