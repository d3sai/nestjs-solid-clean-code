# Unit-тестування та SOLID

## Суть підходу

Архітектура, де залежності передаються через конструктор (DI), ідеально підходить для написання тестів. Для ізоляції логіки сервісу від бази даних чи зовнішніх API використовуються мок-об'єкти (mocks) — найчастіше через Jest.

## Чому DI робить тестування простим

Коли сервіс залежить від абстракції (інтерфейсу/токена), а не створює залежності сам через `new`, у тесті можна підставити мок замість реальної реалізації — без жодних змін у самому класі, що тестується.

```ts
@Injectable()
export class OrderService {
  constructor(private readonly orderRepository: OrderRepository) {}

  async createOrder(dto: CreateOrderDto) {
    if (dto.amount <= 0) {
      throw new BadRequestException('Amount must be positive');
    }
    return this.orderRepository.save(dto);
  }
}
```

```ts
describe('OrderService', () => {
  let service: OrderService;
  let repository: { save: jest.Mock };

  beforeEach(async () => {
    repository = { save: jest.fn() };

    const module = await Test.createTestingModule({
      providers: [
        OrderService,
        { provide: OrderRepository, useValue: repository },
      ],
    }).compile();

    service = module.get(OrderService);
  });

  it('throws when amount is not positive', async () => {
    await expect(
      service.createOrder({ amount: -10 } as CreateOrderDto),
    ).rejects.toThrow(BadRequestException);
  });

  it('saves order via repository', async () => {
    repository.save.mockResolvedValue({ id: 1, amount: 100 });

    const result = await service.createOrder({ amount: 100 } as CreateOrderDto);

    expect(repository.save).toHaveBeenCalledWith({ amount: 100 });
    expect(result).toEqual({ id: 1, amount: 100 });
  });
});
```

Тест перевіряє бізнес-логіку `OrderService` повністю ізольовано — без реальної БД, без мережевих викликів.

## Зв'язок із SOLID

- **DIP** — сервіс залежить від абстракції репозиторію, тому реальну реалізацію легко підмінити мок-об'єктом у тесті (`useValue`).
- **SRP** — коли клас відповідає лише за одну задачу, його простіше й дешевше тестувати: менше причин для зміни тесту, менше побічних ефектів, які треба мокати.
- **ISP** — вузькі інтерфейси означає менше методів, які треба мокати для одного тесту.

<!-- Місце для доповнення: тестування контролерів, мокання ConfigService, e2e-тести через supertest -->
