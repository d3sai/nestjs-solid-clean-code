# OCP — Open-Closed Principle

## The essence of the principle

Code should be open for extension but closed for modification. Adding new functionality shouldn't require changing existing, already-tested code — instead, new behavior is added through new classes that implement a shared abstraction.

## Example in NestJS

A classic case is different payment methods. Instead of a single service with an `if/switch` for every new provider, an abstract class or interface-based strategy is used:

```ts
// ✅ The abstraction that every new strategy conforms to
export interface PaymentStrategy {
  pay(amount: number): Promise<PaymentResult>;
}

@Injectable()
export class StripePaymentStrategy implements PaymentStrategy {
  async pay(amount: number): Promise<PaymentResult> {
    // Stripe logic
  }
}

@Injectable()
export class PaypalPaymentStrategy implements PaymentStrategy {
  async pay(amount: number): Promise<PaymentResult> {
    // PayPal logic
  }
}
```

When a new payment provider appears, a new class implementing `PaymentStrategy` is added. The existing strategies and the code that uses them remain unchanged.

```ts
// ❌ Bad: every new payment method requires editing this same method
@Injectable()
export class PaymentService {
  pay(method: string, amount: number) {
    if (method === 'stripe') { /* ... */ }
    else if (method === 'paypal') { /* ... */ }
    // with every new method the class grows and risks breaking existing behavior
  }
}
```

<!-- Open for expansion: DI tokens for selecting a strategy, an example of Strategy + Factory in a Nest module -->
