# OCP — Open-Closed Principle

## Суть принципу

Код має бути відкритим для розширення, але закритим для модифікації. Додавання нової функціональності не повинно вимагати зміни вже існуючого й протестованого коду — натомість нову поведінку додають через нові класи, що реалізують спільну абстракцію.

## Приклад у NestJS

Класичний кейс — різні методи оплати. Замість одного сервісу з `if/switch` на кожен новий провайдер, використовують абстрактний клас або інтерфейс-стратегію:

```ts
// ✅ Абстракція, під яку підлаштовується кожна нова стратегія
export interface PaymentStrategy {
  pay(amount: number): Promise<PaymentResult>;
}

@Injectable()
export class StripePaymentStrategy implements PaymentStrategy {
  async pay(amount: number): Promise<PaymentResult> {
    // логіка Stripe
  }
}

@Injectable()
export class PaypalPaymentStrategy implements PaymentStrategy {
  async pay(amount: number): Promise<PaymentResult> {
    // логіка PayPal
  }
}
```

Коли з'являється новий провайдер оплати — додається новий клас, що реалізує `PaymentStrategy`. Існуючі стратегії та код, що ними користується, не змінюються.

```ts
// ❌ Погано: кожен новий спосіб оплати вимагає правки цього ж методу
@Injectable()
export class PaymentService {
  pay(method: string, amount: number) {
    if (method === 'stripe') { /* ... */ }
    else if (method === 'paypal') { /* ... */ }
    // з кожним новим методом клас росте і ризикує зламати існуюче
  }
}
```

<!-- Місце для доповнення: DI-токени для вибору стратегії, приклад Strategy + Factory в Nest-модулі -->
