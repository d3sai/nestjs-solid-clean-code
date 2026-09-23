---
note: Розділ додано понад початково узгоджений зміст — Enterprise/Senior-рівень архітектури, за запитом користувача.
---

# CQRS (Command Query Responsibility Segregation)

## Суть підходу

CQRS — поділ операцій на два типи:

- **Commands** — зміна стану (створення, оновлення, видалення).
- **Queries** — читання даних.

У великих проєктах це дозволяє оптимізувати шлях запису й шлях читання окремо — наприклад, різні моделі даних, різні джерела (read-replica для читання), різне кешування, без взаємного впливу одне на одне.

## Приклад у NestJS (`@nestjs/cqrs`)

```ts
// Command — намір змінити стан
export class CreateOrderCommand {
  constructor(public readonly dto: CreateOrderDto) {}
}

@CommandHandler(CreateOrderCommand)
export class CreateOrderHandler implements ICommandHandler<CreateOrderCommand> {
  constructor(private readonly orderRepository: OrderRepository) {}

  async execute(command: CreateOrderCommand) {
    return this.orderRepository.save(command.dto);
  }
}
```

```ts
// Query — намір прочитати дані
export class GetOrderByIdQuery {
  constructor(public readonly id: number) {}
}

@QueryHandler(GetOrderByIdQuery)
export class GetOrderByIdHandler implements IQueryHandler<GetOrderByIdQuery> {
  constructor(private readonly orderReadRepository: OrderReadRepository) {}

  async execute(query: GetOrderByIdQuery) {
    return this.orderReadRepository.findById(query.id);
  }
}
```

```ts
@Controller('orders')
export class OrderController {
  constructor(
    private readonly commandBus: CommandBus,
    private readonly queryBus: QueryBus,
  ) {}

  @Post()
  create(@Body() dto: CreateOrderDto) {
    return this.commandBus.execute(new CreateOrderCommand(dto));
  }

  @Get(':id')
  getById(@Param('id') id: number) {
    return this.queryBus.execute(new GetOrderByIdQuery(id));
  }
}
```

Контролер не викликає сервіс напряму — він лише диспетчерить команди/запити через `CommandBus`/`QueryBus`, а кожен handler відповідає за одну конкретну операцію (це також SRP у чистому вигляді).

## Коли це виправдано, а коли — overhead

CQRS додає суттєву складність (окремі handler-и, окремі моделі читання/запису), тож він виправданий у великих системах з високим навантаженням на читання чи запис окремо. Для типового CRUD-застосунку це, як і надмірний Repository Pattern (див. `references/data-validation/repository-pattern.md`), швидше overhead, ніж користь — варто зважувати через призму YAGNI (`references/principles/yagni.md`).

<!-- Місце для доповнення: Event Sourcing у парі з CQRS, приклад окремих read/write моделей БД -->
