# Анти-патерни Dependency Injection

Dependency Injection — потужний патерн, але його неправильне використання руйнує архітектуру застосунку не менше, ніж повна його відсутність. Основні анти-патерни:

## 1. Service Locator (прихована залежність)

Замість ін'єкції конкретних залежностей через конструктор, у клас передається весь `ModuleRef`/`Injector`, і потрібний сервіс "дістається" вручну під час виконання.

```ts
// ❌ Погано: залежності класу неявні, їх не видно з конструктора
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
// ✅ Добре: явна залежність через конструктор
@Injectable()
export class OrderService {
  constructor(private readonly paymentService: PaymentService) {}

  async createOrder(dto: CreateOrderDto) {
    return this.paymentService.charge(dto.amount);
  }
}
```

Наслідок анти-патерну: неможливо зрозуміти, від чого залежить клас, не читаючи всю реалізацію методів, а підміна залежності в тесті перетворюється на боротьбу з `ModuleRef` замість простого `useValue`.

## 2. Circular Dependencies (циклічні залежності)

Сервіс А залежить від сервісу Б, а сервіс Б — від сервісу А. У NestJS це часто призводить до помилок ініціалізації (одна із залежностей приходить `undefined`).

```ts
// ❌ UsersService і OrdersService залежать один від одного напряму
@Injectable()
export class UsersService {
  constructor(private readonly ordersService: OrdersService) {}
}

@Injectable()
export class OrdersService {
  constructor(private readonly usersService: UsersService) {}
}
```

Швидке рішення — `forwardRef()`:

```ts
@Injectable()
export class UsersService {
  constructor(
    @Inject(forwardRef(() => OrdersService))
    private readonly ordersService: OrdersService,
  ) {}
}
```

Але `forwardRef()` лише прибирає симптом. Правильне рішення — переглянути дизайн: винести спільну логіку в третій сервіс, від якого залежать обидва, або замінити прямий виклик на подію (див. `references/architecture/event-driven-architecture.md`), щоб розірвати цикл узагалі.

## 3. DI Container Abuse (перевантаження контейнера)

DI використовується для передачі конфігурацій, констант чи довільних об'єктів, які не є повноцінними сервісами.

```ts
// ❌ Погано: контейнер перетворюється на "смітник" довільних значень
@Module({
  providers: [
    { provide: 'API_KEY', useValue: 'sk_live_...' },
    { provide: 'MAX_RETRIES', useValue: 3 },
    { provide: 'FEATURE_FLAG_NEW_CHECKOUT', useValue: true },
  ],
})
export class AppModule {}
```

Для налаштувань є спеціалізований інструмент — `ConfigService` (див. `references/architecture/configuration.md`), а не ін'єкція окремого провайдера на кожне значення:

```ts
// ✅ Добре: усі налаштування — через єдину точку входу
@Injectable()
export class PaymentService {
  constructor(private readonly configService: ConfigService) {}

  private get apiKey() {
    return this.configService.get<string>('API_KEY');
  }
}
```

## 4. Constructor Bloat (занадто багато залежностей)

Якщо конструктор приймає 10+ сервісів — це ознака порушення Single Responsibility Principle (див. `references/solid/srp.md`). Клас робить занадто багато.

```ts
// ❌ Погано: клас явно відповідає за забагато різних речей
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

Рішення — розбити клас на менші, кожен зі своєю вузькою відповідальністю (наприклад, оркестрацію нотифікацій винести в окремий `OrderNotificationService`, який сам залежить від `EmailService`/`SmsService`/`NotificationService`), а не тримати всі 10 залежностей в одному місці.

## 5. Injection of Fat Objects (ін'єкція "важких" об'єктів)

Ін'єктування цілого модуля чи громіздкого об'єкта замість вузькоспеціалізованого сервісу. Залежність повинна бути максимально абстрактною та легкою — це той самий принцип, що й в ISP (див. `references/solid/isp.md`), тільки застосований не до інтерфейсу, а до того, що саме інжектується.

```ts
// ❌ Погано: клас залежить від "товстого" об'єкта, а використовує з нього одне поле
@Injectable()
export class InvoiceService {
  constructor(private readonly appConfig: AppConfig) {} // весь конфіг застосунку

  generate() {
    const currency = this.appConfig.billing.invoice.defaultCurrency;
    // ...
  }
}
```

```ts
// ✅ Добре: залежність — вузька і сфокусована на тому, що реально потрібно
@Injectable()
export class InvoiceService {
  constructor(private readonly configService: ConfigService) {}

  generate() {
    const currency = this.configService.get<string>('DEFAULT_CURRENCY');
  }
}
```

## Чому всі ці анти-патерни шкідливі

- **Складне тестування.** Приховані чи "важкі" залежності неможливо легко підмінити моком — доводиться мокати цілі об'єкти чи весь `ModuleRef` замість одного вузького інтерфейсу.
- **Висока зв'язність (coupling).** Компоненти стають нерозривно пов'язаними, що заважає їх перевикористанню і незалежній зміні.
- **Нечитабельність.** Неможливо зрозуміти, що саме потрібно класу для роботи, просто поглянувши на конструктор — а це головна перевага DI, яку ці анти-патерни якраз і знищують.

<!-- Місце для доповнення: як лінтер (eslint-plugin) може ловити частину цих анти-патернів автоматично, приклад рефакторингу Constructor Bloat крок за кроком -->
