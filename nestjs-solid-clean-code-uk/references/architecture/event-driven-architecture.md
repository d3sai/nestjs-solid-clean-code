---
note: Розділ додано понад початково узгоджений зміст — Enterprise/Senior-рівень архітектури, за запитом користувача.
---

# Event-Driven Architecture (EDA)

## Суть підходу

Замість того, щоб один сервіс напряму викликав інший (наприклад, `OrderService` напряму викликає `EmailService`), сервіс публікує подію (наприклад, `OrderCreatedEvent`). Інші модулі (`Email`, `Analytics`) підписуються на цю подію і реагують незалежно один від одного.

Це робить систему декоплленою (слабкозв'язаною) — модуль-видавець події нічого не знає про те, хто саме її слухає і скільки таких слухачів.

## Приклад у NestJS (`@nestjs/event-emitter`)

Реалізація складається з трьох кроків:

### 1. Реєстрація модуля

`EventEmitterModule` підключається один раз у кореневому модулі застосунку:

```ts
@Module({
  imports: [EventEmitterModule.forRoot()],
})
export class AppModule {}
```

### 2. Публікація та підписка на подію

```ts
// ❌ Пряме зв'язування: OrderService знає про всіх своїх "споживачів"
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
// ✅ Event-driven: OrderService лише повідомляє про факт, що сталося
@Injectable()
export class OrderService {
  constructor(
    private readonly orderRepository: OrderRepository,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async createOrder(dto: CreateOrderDto) {
    const order = await this.orderRepository.save(dto);
    // Публікація події — EventEmitter2 після завершення операції
    this.eventEmitter.emit('order.created', new OrderCreatedEvent(order));
    return order;
  }
}
```

### 3. Підписка на подію

Декоратор `@OnEvent` автоматично викликає метод-обробник, коли подію було випущено — підписка не потребує ручної реєстрації, достатньо самого декоратора на методі провайдера.

```ts
@Injectable()
export class EmailListener {
  @OnEvent('order.created')
  async handleOrderCreated(event: OrderCreatedEvent) {
    // відправка листа — Email-модуль сам вирішує, що робити з подією
  }
}

@Injectable()
export class AnalyticsListener {
  @OnEvent('order.created')
  async handleOrderCreated(event: OrderCreatedEvent) {
    // трекінг метрики — незалежно від EmailListener
  }
}
```

## Чому це важливо

- **Слабке зв'язування.** `OrderService` можна змінювати, тестувати й деплоїти незалежно від того, скільки модулів слухають подію `order.created`. Додавання нового слухача (наприклад, `LoyaltyPointsListener`) не вимагає жодної зміни в `OrderService`.
- **Розширюваність (OCP).** Нова реакція на подію — це новий listener, а не правка існуючого сервіса. Див. `references/solid/ocp.md`.
- **Межі модулів.** Це один з інструментів для комунікації між Bounded Contexts без прямих залежностей — див. `references/architecture/module-boundaries.md`.

## Компроміс

Event-driven підхід ускладнює трасування потоку виконання: щоб зрозуміти, що відбувається після `order.created`, потрібно знайти всіх слухачів по всьому коду — на відміну від прямого виклику методу, де шлях виконання очевидний одразу. Для простих, невеликих застосунків прямий виклик сервісу часто читабельніший і достатній.

<!-- Місце для доповнення: async vs sync events, обробка помилок у listener-ах, event bus між мікросервісами (Kafka/RabbitMQ) vs internal EventEmitter -->
