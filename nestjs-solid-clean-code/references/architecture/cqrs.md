---
note: Section added beyond the originally agreed table of contents — Enterprise/Senior-level architecture, added at the user's request.
---

# CQRS (Command Query Responsibility Segregation)

## Core Idea

CQRS splits operations into two types:

- **Commands** — change state (create, update, delete).
- **Queries** — read data.

In large projects, this lets you optimize the write path and the read path separately — for example, different data models, different sources (a read replica for reads), different caching, without one affecting the other.

## Example in NestJS (`@nestjs/cqrs`)

```ts
// Command — an intent to change state
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
// Query — an intent to read data
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

The controller doesn't call the service directly — it only dispatches commands/queries through the `CommandBus`/`QueryBus`, and each handler is responsible for exactly one operation (this is also SRP in its pure form).

## When It's Justified and When It's Overhead

CQRS adds substantial complexity (separate handlers, separate read/write models), so it's justified in large systems with high load on reads or writes handled separately. For a typical CRUD application, it — much like an overused Repository Pattern (see `references/data-validation/repository-pattern.md`) — tends to be overhead rather than a benefit, and should be weighed through the lens of YAGNI (`references/principles/yagni.md`).

<!-- Open for expansion: Event Sourcing paired with CQRS, an example of separate read/write database models -->
