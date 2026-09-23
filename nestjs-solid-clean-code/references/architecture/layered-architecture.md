# Layered Architecture

## Core Idea

Database logic must not be mixed with business logic. The classic structure for NestJS:

```
Controller (incoming data) → Service (business processes) → Repository/Data Access (database interaction)
```

Each layer is responsible for its own concern and doesn't "peek" into the details of the neighboring layer.

- **Controller** — accepts the HTTP request, validates incoming data (via a DTO), calls the appropriate service, and returns the response. No business logic and no direct database queries.
- **Service** — business processes, rules, orchestration. Knows nothing about SQL, the ORM, or HTTP — it works through the repository abstraction.
- **Repository / Data Access** — the single place that knows exactly how data is stored and retrieved (ORM, raw SQL queries, an external API, etc.).

```ts
@Controller('orders')
export class OrderController {
  constructor(private readonly orderService: OrderService) {}

  @Post()
  create(@Body() dto: CreateOrderDto) {
    return this.orderService.createOrder(dto);
  }
}

@Injectable()
export class OrderService {
  constructor(private readonly orderRepository: OrderRepository) {}

  async createOrder(dto: CreateOrderDto) {
    // business rules: validation, calculations, calls to other services
    return this.orderRepository.save(dto);
  }
}

@Injectable()
export class OrderRepository {
  constructor(
    @InjectRepository(OrderEntity) private readonly repo: Repository<OrderEntity>,
  ) {}

  save(data: CreateOrderDto) {
    return this.repo.save(data);
  }
}
```

## Why This Matters

This separation makes it easy to swap out the ORM (for example, migrating from Prisma to TypeORM) without touching the business rules — only the Repository/Data Access layer needs to be rewritten. `Service` and `Controller` don't even "find out" about the change, because they work through the same contract.

This is the same principle as in `references/solid/dip.md` — business logic depends on the data access abstraction, not on a concrete ORM implementation.

<!-- Open for expansion: an example with a Repository interface (DIP in its pure form), showing exactly where the layer boundary sits in a Nest module -->
