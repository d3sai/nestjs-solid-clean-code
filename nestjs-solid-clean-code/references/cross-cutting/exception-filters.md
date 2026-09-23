# Global Error Handling (Exception Filters)

## Core Idea

You shouldn't duplicate `try-catch` in every service method. Instead, create a global error filter (`ExceptionFilter`) that catches exceptions in one place and converts them into standardized HTTP responses.

## The Problem Without a Filter

```ts
// ❌ Bad: try-catch is duplicated in every method
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

## Solution: a Global Exception Filter

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
// Global registration — a single place for the whole application
app.useGlobalFilters(new GlobalExceptionFilter());
```

Now the services stay clean — they simply throw semantically meaningful exceptions, and converting them into an HTTP response is the filter's responsibility:

```ts
// ✅ Good: the service knows nothing about HTTP, it just throws a domain exception
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

## Why This Matters

This is a direct application of SRP at the whole-application level: error handling is a separate responsibility, extracted from the services' business logic into a dedicated component. The service code becomes significantly cleaner and easier to read.

<!-- Open for expansion: custom domain exceptions (e.g., DomainException), separate filters for different error types, logging within the filter -->
