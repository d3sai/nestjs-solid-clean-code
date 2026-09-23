# Глобальна обробка помилок (Exception Filters)

## Суть підходу

Не варто дублювати `try-catch` у кожному методі сервісу. Замість цього створюється глобальний фільтр помилок (`ExceptionFilter`), який перехоплює винятки в одному місці та перетворює їх на стандартизовані HTTP-відповіді.

## Проблема без фільтра

```ts
// ❌ Погано: try-catch дублюється в кожному методі
@Injectable()
export class OrderService {
  async createOrder(dto: CreateOrderDto) {
    try {
      return await this.orderRepository.save(dto);
    } catch (err) {
      throw new HttpException('Failed to create order', 500);
    }
  }

  async cancelOrder(id: number) {
    try {
      return await this.orderRepository.cancel(id);
    } catch (err) {
      throw new HttpException('Failed to cancel order', 500);
    }
  }
}
```

## Рішення: глобальний Exception Filter

```ts
@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();

    const status =
      exception instanceof HttpException ? exception.getStatus() : 500;

    const message =
      exception instanceof HttpException
        ? exception.getResponse()
        : 'Internal server error';

    response.status(status).json({
      statusCode: status,
      message,
      timestamp: new Date().toISOString(),
    });
  }
}
```

```ts
// Реєстрація глобально — одне місце для всього застосунку
app.useGlobalFilters(new GlobalExceptionFilter());
```

Тепер сервіси лишаються чистими — вони просто кидають (throw) семантично зрозумілі винятки, а перетворення в HTTP-відповідь — відповідальність фільтра:

```ts
// ✅ Добре: сервіс не знає про HTTP, просто кидає доменний виняток
@Injectable()
export class OrderService {
  async createOrder(dto: CreateOrderDto) {
    const order = await this.orderRepository.save(dto);
    if (!order) {
      throw new NotFoundException('Order could not be created');
    }
    return order;
  }
}
```

## Чому це важливо

Це пряме застосування SRP на рівні всього застосунку: обробка помилок — окрема відповідальність, винесена з бізнес-логіки сервісів у спеціалізований компонент. Код сервісів стає значно чистішим і легшим для читання.

<!-- Місце для доповнення: кастомні доменні винятки (наприклад, DomainException), окремі фільтри для різних типів помилок, логування в фільтрі -->
