# Unit Testing and SOLID

## Core Idea

An architecture where dependencies are passed through the constructor (DI) is ideal for writing tests. Mock objects (mocks) — most often via Jest — are used to isolate a service's logic from the database or external APIs.

## Why DI Makes Testing Easy

When a service depends on an abstraction (an interface/token) instead of creating its dependencies itself with `new`, a test can substitute a mock for the real implementation — with no changes to the class under test.

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

The test verifies `OrderService`'s business logic in complete isolation — no real database, no network calls.

## Connection to SOLID

- **DIP** — the service depends on the repository abstraction, so the real implementation is easy to swap for a mock object in the test (`useValue`).
- **SRP** — when a class is responsible for only one task, it's simpler and cheaper to test: fewer reasons for the test to change, fewer side effects that need to be mocked.
- **ISP** — narrow interfaces mean fewer methods need to be mocked for a single test.

<!-- Open for expansion: testing controllers, mocking ConfigService, e2e tests via supertest -->
