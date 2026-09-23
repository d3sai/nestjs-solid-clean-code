# Layered Architecture (Багаторівнева архітектура)

## Суть підходу

Логіку роботи з БД не можна змішувати з бізнес-логікою. Класична структура для NestJS:

```
Controller (вхідні дані) → Service (бізнес-процеси) → Repository/Data Access (взаємодія з БД)
```

Кожен шар відповідає за своє й не «підглядає» в деталі сусіднього шару.

- **Controller** — приймає HTTP-запит, валідує вхідні дані (через DTO), викликає потрібний сервіс і повертає відповідь. Жодної бізнес-логіки чи прямих запитів до БД.
- **Service** — бізнес-процеси, правила, оркестрація. Не знає про SQL, ORM чи HTTP — працює через абстракцію репозиторію.
- **Repository / Data Access** — єдине місце, що знає, як саме дані зберігаються і дістаються (ORM, сирі SQL-запити, зовнішнє API тощо).

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
    // бізнес-правила: перевірки, розрахунки, виклики інших сервісів
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

## Чому це важливо

Такий поділ дозволяє легко замінити ORM (наприклад, переїхати з Prisma на TypeORM), не зачіпаючи бізнес-правила — досить переписати лише шар Repository/Data Access. `Service` і `Controller` про цю зміну навіть не «дізнаються», бо працюють через той самий контракт.

Це той самий принцип, що і в `references/solid/dip.md` — бізнес-логіка залежить від абстракції доступу до даних, а не від конкретної реалізації ORM.

<!-- Місце для доповнення: приклад з інтерфейсом Repository (DIP у чистому вигляді), де саме межа шарів у Nest-модулі -->
