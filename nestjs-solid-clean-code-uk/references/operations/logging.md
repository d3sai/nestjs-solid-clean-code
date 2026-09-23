# Структуроване логування

## Суть підходу

`console.log` достатньо для локальної розробки, але в проді логи потрібно шукати, фільтрувати й агрегувати — а це вимагає структурованого (JSON) формату, рівнів важливості та наскрізного ідентифікатора запиту.

## Structured logging через Pino

`nestjs-pino` замінює вбудований логер Nest на структурований, з мінімальним оверхедом продуктивності:

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
    // JSON-вивід: { "level": "info", "orderId": "...", "amount": ..., "msg": "Order created" }
  }
}
```

Структурований лог (замість рядка тексту) дозволяє шукати й фільтрувати за полями (`orderId`, `userId`) у системі агрегації логів, а не парсити текст регулярками.

## Correlation ID (Request ID)

Один HTTP-запит часто проходить через кілька сервісів чи внутрішніх викликів. Без наскрізного ідентифікатора неможливо зібрати всі логи, що стосуються одного запиту, докупи.

```ts
// Middleware, що генерує/пробрасує request ID
@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    req['requestId'] = req.headers['x-request-id'] ?? randomUUID();
    next();
  }
}
```

Далі цей ідентифікатор передається у всі логи в межах запиту — найзручніше через `AsyncLocalStorage` (наприклад, `nestjs-cls`), щоб не прокидати його вручну через кожен виклик функції:

```ts
this.logger.log({ requestId: this.cls.get('requestId'), orderId: order.id }, 'Order created');
```

## Рівні логів

- **error** — щось зламалось, потребує уваги (виняток, недоступна залежність).
- **warn** — потенційна проблема, застосунок продовжує працювати (застарілий API, повільний запит).
- **info** — важливі бізнес-події (замовлення створено, користувач зареєструвався).
- **debug** — деталі для діагностики, вимикаються в проді за замовчуванням.

Логування "всього підряд" на рівні `info` так само шкідливе, як і брак логів — важливий сигнал губиться в шумі.

## Що НІКОЛИ не можна логувати

Паролі, токени доступу, номери карток, персональні дані користувачів — те саме, що не можна повертати в response (`references/security/security-hardening.md`). Помилка тут — типовий шлях витоку чутливих даних, бо логи часто зберігаються довше і мають ширший доступ, ніж сама БД.

```ts
// ❌ Небезпечно: весь DTO логується "як є", разом із паролем
this.logger.log({ dto }, 'Registration attempt');
```

```ts
// ✅ Безпечно: логується лише те, що дійсно потрібно для діагностики
this.logger.log({ email: dto.email }, 'Registration attempt');
```

## Чому це важливо

Централізоване, структуроване логування — це те, що робить `references/operations/health-monitoring.md` (Health Checks, Prometheus/Grafana) насправді придатним для діагностики: метрики показують, ЩО зламалось, а логи з correlation ID — ЧОМУ саме.

<!-- Місце для доповнення: агрегація логів (ELK/Loki), інтеграція з розподіленим трейсингом (OpenTelemetry) -->
