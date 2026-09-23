# Idempotency

## Core Idea

An idempotent operation produces the same result no matter how many times it's repeated. This is critical for operations that can be invoked again through no fault of the client — a network drop, a timeout, a payment provider re-delivering a webhook.

## The Problem Without Idempotency

```ts
// ❌ Dangerous: if the client didn't get a response due to a timeout and retried the request,
// the order (and the charge) gets created twice
@Post('orders')
async createOrder(@Body() dto: CreateOrderDto) {
  return this.orderService.createOrder(dto);
}
```

A network failure doesn't mean the operation didn't complete on the server — it may well have succeeded, with the client simply never receiving the confirmation. A naive retry on the client then duplicates the operation.

## Solution: Idempotency Key

The client generates a unique key for a specific attempt at the operation (for example, a UUID) and passes it in a header. The server remembers the result for that key, and on a repeated request with the same key, returns the stored result instead of executing the operation again:

```ts
@Injectable()
export class IdempotencyInterceptor implements NestInterceptor {
  constructor(@Inject(CACHE_MANAGER) private readonly cache: Cache) {}

  async intercept(context: ExecutionContext, next: CallHandler): Promise<Observable<any>> {
    const request = context.switchToHttp().getRequest();
    const key = request.headers['idempotency-key'];
    if (!key) return next.handle();

    const cached = await this.cache.get(`idempotency:${key}`);
    if (cached) return of(cached); // return the stored result instead of running the operation again

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

## Where This Is Especially Critical

- **Payment operations.** A double charge caused by a repeated webhook or retry is one of the most expensive bugs a payment system can have.
- **Webhook handlers.** Payment providers (Stripe, PayPal) may send the same webhook multiple times — the handler must recognize a duplicate by the provider's event ID and avoid processing it twice.
- **Any operation that debits a resource** (awarding bonus points, drawing down a limit) — anywhere double execution directly means financial loss or inconsistent data.

```ts
// Idempotency at the webhook-handler level, via the provider's event ID
@Post('webhooks/stripe')
async handleStripeWebhook(@Body() event: StripeEvent) {
  const alreadyProcessed = await this.webhookLogRepository.exists(event.id);
  if (alreadyProcessed) return { status: 'already processed' };

  await this.webhookLogRepository.save({ id: event.id });
  await this.paymentService.handleEvent(event);
}
```

### Idempotency by `event.id` Is Necessary but Not Sufficient

Recognizing a duplicate by `event.id` protects against reprocessing **the same** event twice. But for Stripe (and similar payment providers), the event type alone is often not enough to determine whether the operation actually completed successfully:

```ts
// ❌ Incomplete check: the event type alone doesn't guarantee the money has actually arrived
case 'checkout.session.completed': {
  await this.orderService.markAsPaid(orderId); // premature for async payment methods!
}
```

For asynchronous payment methods (for example, some bank transfers), the `checkout.session.completed` event can arrive with the session status set to `completed`, while the `payment_status` field is still `unpaid`: the payment itself will only be confirmed by a separate, later event.

```ts
// ✅ Check not just the event type, but the actual payment status inside it
case 'checkout.session.completed': {
  const session = event.data.object as Stripe.Checkout.Session;
  if (session.payment_status === 'paid') {
    await this.orderService.markAsPaid(orderId);
  }
  // if payment_status !== 'paid' — wait for the separate async_payment_succeeded event
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

This is a more general lesson than Stripe itself: idempotency (not processing an event twice) and correctness (correctly understanding whether the event actually means "success") are two separate concerns, and both need to be handled. Always check the specific provider's documentation for which payload fields actually confirm that an operation completed, and which merely report an intermediate state.

## Why This Matters

GET requests are idempotent by nature (reading again changes nothing), but POST/PATCH are not, unless the developer deliberately makes them so. In distributed systems (queues, webhooks, retries at the HTTP client level), redelivery isn't a rare edge case — it's an expected norm that critical operations should be designed for from the start.

<!-- Open for expansion: idempotency at the database level via a unique constraint instead of a cache, TTL for idempotency keys -->
