---
note: Section added beyond the originally agreed table of contents — Enterprise/Senior-level architecture, added at the user's request.
---

# Database Transactions (Unit of Work)

## Core Idea

If an operation touches multiple tables, it's important to guarantee data integrity: either all the changes apply, or none of them do. Without a transaction, you can end up in a situation where the write to one table succeeds while the write to a related one doesn't, leaving the data in an inconsistent state.

## Example in NestJS with Prisma

```ts
@Injectable()
export class OrderService {
  constructor(private readonly prisma: PrismaService) {}

  async createOrderWithPayment(dto: CreateOrderDto) {
    // if any step fails, both records are rolled back
    return this.prisma.$transaction(async (tx) => {
      const order = await tx.order.create({ data: dto });
      await tx.payment.create({
        data: { orderId: order.id, amount: dto.amount, status: 'pending' },
      });
      return order;
    });
  }
}
```

## Example in NestJS with TypeORM

```ts
@Injectable()
export class OrderService {
  constructor(private readonly dataSource: DataSource) {}

  async createOrderWithPayment(dto: CreateOrderDto) {
    return this.dataSource.transaction(async (manager) => {
      const order = await manager.save(OrderEntity, dto);
      await manager.save(PaymentEntity, { orderId: order.id, amount: dto.amount });
      return order;
    });
  }
}
```

## Why This Matters

- **Data integrity.** A transaction guarantees atomicity — an operation spanning multiple tables executes as a single whole.
- **Boundary of responsibility.** Transaction logic is a detail of working with the database, so it belongs in the data layer (the Repository/Service that talks directly to the store), not scattered across higher-level business logic. This aligns with `references/architecture/layered-architecture.md`.

## What to Watch Out For

Avoid "threading" the transaction object (`tx`/`manager`) through too many layers of abstraction — if the repository is fully isolated from the ORM (a pure Repository Pattern), passing the transactional context through it becomes cumbersome. In such cases, it's often more pragmatic to keep the transaction logic right in the service that orchestrates several repositories, rather than trying to keep the abstraction "pure" at any cost — this is the same trade-off described in `references/data-validation/repository-pattern.md` (when the Repository Pattern becomes overhead).

<!-- Open for expansion: the @Transactional() decorator (cls-hooked / nestjs-cls), distributed transactions across microservices (the Saga pattern) -->
