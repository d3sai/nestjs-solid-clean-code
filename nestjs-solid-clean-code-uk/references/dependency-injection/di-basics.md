# Dependency Injection в NestJS: основи

## Суть механізму

Dependency Injection (DI) — вбудований у NestJS механізм керування залежностями між класами. Замість того, щоб клас сам створював свої залежності (`new SomeService()`), він лише декларує, що йому потрібно, а фреймворк сам створює потрібний екземпляр і передає його.

## Три кроки для реалізації DI

### 1. Створення сервісу

Клас позначається декоратором `@Injectable()` — це робить його «провайдером», який Nest може створити й ін'єктувати в інші класи.

```ts
@Injectable()
export class PaymentService {
  process() {
    /* логіка обробки платежу */
  }
}
```

### 2. Реєстрація у модулі

Клас додається в масив `providers` відповідного модуля, щоб Nest знав про його існування і міг створити для нього екземпляр (Nest будує граф залежностей саме на основі цього масиву).

```ts
@Module({
  providers: [PaymentService],
})
export class OrderModule {}
```

### 3. Ін'єкція через конструктор

Потрібний сервіс просто вказується в конструкторі контролера чи іншого сервісу — Nest автоматично створює екземпляр і підставляє його.

```ts
@Controller('orders')
export class OrderController {
  constructor(private readonly paymentService: PaymentService) {}
}
```

## Чому це важливо

- **Тестованість.** Реальний сервіс легко підмінити «моком» під час тестування — не потрібно змінювати сам клас, що споживає залежність, достатньо підмінити провайдер у тестовому модулі.
- **Loose coupling (слабке зв'язування).** Класи не створюють свої залежності самостійно через `new`. Це напряму реалізує **Dependency Inversion Principle (DIP)**: компоненти залежать від абстракцій, а не від конкретних реалізацій (див. `references/solid/dip.md`).

## DI через токени: не лише класи, а й абстракції

DI в Nest не обмежується конкретними класами — можна ін'єктувати й інтерфейси або абстрактні класи за допомогою токенів і декоратора `@Inject()`. Це робить систему гнучкою для заміни стратегій «на льоту» — наприклад, заміни одного платіжного шлюзу на інший без зміни коду, що ним користується.

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

Заміна `StripeGateway` на інший платіжний шлюз відбувається зміною одного рядка (`useClass`) у модулі — без правок в `OrderService`. Детальніше про типові помилки, коли DI ламають (наприклад, `new Service()` замість ін'єкції) — у `references/dependency-injection/di-anti-patterns.md`.

<!-- Місце для доповнення: useFactory, useValue, scope (REQUEST/TRANSIENT), circular dependency через forwardRef -->
