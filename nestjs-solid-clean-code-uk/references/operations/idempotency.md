# Ідемпотентність

## Суть підходу

Ідемпотентна операція дає той самий результат незалежно від того, скільки разів її повторили. Це критично для операцій, які можуть бути викликані повторно не з вини клієнта — обрив мережі, таймаут, повторна доставка webhook від платіжної системи.

## Проблема без ідемпотентності

```ts
// ❌ Небезпечно: якщо клієнт не отримав відповідь через таймаут і повторив запит,
// замовлення (і списання коштів) створиться двічі
@Post('orders')
async createOrder(@Body() dto: CreateOrderDto) {
  return this.orderService.createOrder(dto);
}
```

Мережевий збій не означає, що операція на сервері не виконалась — вона могла успішно завершитись, а клієнт просто не отримав підтвердження. Наївний retry на клієнті в такому разі дублює операцію.

## Рішення: Idempotency Key

Клієнт генерує унікальний ключ для конкретної спроби операції (наприклад, UUID) і передає його в заголовку. Сервер запам'ятовує результат для цього ключа й при повторному запиті з тим самим ключем повертає збережений результат, не виконуючи операцію повторно:

```ts
@Injectable()
export class IdempotencyInterceptor implements NestInterceptor {
  constructor(@Inject(CACHE_MANAGER) private readonly cache: Cache) {}

  async intercept(context: ExecutionContext, next: CallHandler): Promise<Observable<any>> {
    const request = context.switchToHttp().getRequest();
    const key = request.headers['idempotency-key'];
    if (!key) return next.handle();

    const cached = await this.cache.get(`idempotency:${key}`);
    if (cached) return of(cached); // повертаємо збережений результат, не виконуючи операцію знову

    return next.handle().pipe(
      tap(async (result) => {
        await this.cache.set(`idempotency:${key}`, result, { ttl: 86400 });
      }),
    );
  }
}
```

```ts
@UseInterceptors(IdempotencyInterceptor)
@Post('orders')
createOrder(@Body() dto: CreateOrderDto) {
  return this.orderService.createOrder(dto);
}
```

## Де це особливо критично

- **Платіжні операції.** Подвійне списання коштів через повторний webhook чи ретрай — один із найдорожчих можливих багів у платіжній системі.
- **Webhook-обробники.** Платіжні провайдери (Stripe, PayPal) можуть надіслати той самий webhook кілька разів — обробник повинен розпізнати дублікат за ID події провайдера й не обробляти його вдруге.
- **Будь-яка операція зі списанням ресурсу** (нарахування бонусів, списання ліміту) — там, де подвійне виконання напряму означає фінансові втрати чи неконсистентні дані.

```ts
// Ідемпотентність на рівні webhook-обробника через ID події провайдера
@Post('webhooks/stripe')
async handleStripeWebhook(@Body() event: StripeEvent) {
  const alreadyProcessed = await this.webhookLogRepository.exists(event.id);
  if (alreadyProcessed) return { status: 'already processed' };

  await this.webhookLogRepository.save({ id: event.id });
  await this.paymentService.handleEvent(event);
}
```

### Ідемпотентність за `event.id` — необхідна, але не достатня умова

Розпізнавання дубліката за `event.id` захищає від повторної обробки **однієї й тієї самої** події. Але для Stripe (і подібних платіжних провайдерів) типу події самого по собі часто недостатньо, щоб зрозуміти, чи операція дійсно завершилась успішно:

```ts
// ❌ Неповна перевірка: тип події ще не гарантує, що гроші реально надійшли
case 'checkout.session.completed': {
  await this.orderService.markAsPaid(orderId); // передчасно для async-методів оплати!
}
```

Для асинхронних методів оплати (наприклад, деякі банківські перекази) подія `checkout.session.completed` може прийти зі статусом сесії `completed`, а поле `payment_status` — усе ще `unpaid`: сама оплата підтвердиться окремою подією пізніше.

```ts
// ✅ Перевіряємо не лише тип події, а й фактичний статус оплати всередині неї
case 'checkout.session.completed': {
  const session = event.data.object as Stripe.Checkout.Session;
  if (session.payment_status === 'paid') {
    await this.orderService.markAsPaid(orderId);
  }
  // якщо payment_status !== 'paid' — чекаємо окрему подію async_payment_succeeded
  break;
}

case 'checkout.session.async_payment_succeeded': {
  await this.orderService.markAsPaid(orderId);
  break;
}

case 'checkout.session.async_payment_failed': {
  await this.orderService.markAsFailed(orderId);
  break;
}
```

Це загальніший урок, ніж сам Stripe: ідемпотентність (не обробити подію двічі) і коректність (правильно зрозуміти, чи подія взагалі означає "успіх") — дві різні задачі, і закривати треба обидві. Завжди звіряйтесь із документацією конкретного провайдера, які поля в payload насправді підтверджують завершення операції, а які лише повідомляють про проміжний стан.

## Чому це важливо

GET-запити ідемпотентні за своєю природою (повторне читання нічого не змінює), але POST/PATCH — ні, доки розробник свідомо не забезпечить ідемпотентність. У розподілених системах (черги, webhook, ретраї на рівні HTTP-клієнта) повторна доставка — це не рідкісний edge case, а очікувана норма, під яку варто проєктувати критичні операції із самого початку.

<!-- Місце для доповнення: ідемпотентність на рівні БД через unique constraint замість кешу, TTL для idempotency-ключів -->
