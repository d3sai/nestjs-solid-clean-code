---
note: Розділ додано понад початково узгоджений зміст — Enterprise/Senior-рівень архітектури, за запитом користувача.
---

# Database Transactions (Unit of Work)

## Суть підходу

Якщо операція зачіпає кілька таблиць, важливо забезпечити цілісність даних: або всі зміни застосовуються, або жодна. Без транзакції можлива ситуація, коли запис в одній таблиці пройшов, а в пов'язаній — ні, і дані лишаються в неконсистентному стані.

## Приклад у NestJS з Prisma

```ts
@Injectable()
export class OrderService {
  constructor(private readonly prisma: PrismaService) {}

  async createOrderWithPayment(dto: CreateOrderDto) {
    // якщо будь-який крок впаде — обидва записи відкотяться
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

## Приклад у NestJS з TypeORM

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

## Чому це важливо

- **Цілісність даних.** Транзакція гарантує атомарність — операція над кількома таблицями виконується як єдине ціле.
- **Межа відповідальності.** Логіка транзакції — це деталь роботи з БД, тому їй місце в шарі даних (Repository/Service, що напряму працює зі сховищем), а не розмазана по бізнес-логіці вищого рівня. Узгоджується з `references/architecture/layered-architecture.md`.

## На що звернути увагу

Не варто "прокидати" об'єкт транзакції (`tx`/`manager`) через забагато шарів абстракції — якщо репозиторій повністю ізольований від ORM (чистий Repository Pattern), передача транзакційного контексту через нього ускладнюється. У таких випадках часто прагматичніше тримати транзакційну логіку прямо в сервісі, що оркеструє кілька репозиторіїв, ніж намагатись зберегти абстракцію "чистою" за будь-яку ціну — це той самий компроміс, що описаний у `references/data-validation/repository-pattern.md` (коли Repository Pattern стає overhead).

<!-- Місце для доповнення: @Transactional() декоратор (cls-hooked / nestjs-cls), розподілені транзакції між мікросервісами (Saga pattern) -->
