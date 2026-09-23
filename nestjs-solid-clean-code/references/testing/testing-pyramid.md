# Testing Pyramid: Unit, Integration, E2E

## Core Idea

Not all tests are equally cheap and fast. The Testing Pyramid is a model showing the proportions in which you should write tests at different levels: many fast unit tests at the base, fewer integration tests, and only a handful of slow e2e tests at the top.

```
        /\
       /e2e\        few, slow, expensive to maintain
      /------\
     /integr. \     moderate, medium speed
    /----------\
   /  unit tests \  many, fast, cheap
  /----------------\
```

## Unit Tests

Verify a single unit of logic (a service, a function) in complete isolation, with all dependencies mocked. A detailed example is in `references/testing/testing-with-solid.md`. Fast (milliseconds), so there should be hundreds of them — they catch regressions in business logic at the cheapest level.

## Integration Tests

Verify the interaction of several real components together — for example, a service and a real (test) database, without mocking the repository. The goal is to confirm that the ORM queries, transactions (`references/architecture/transactions.md`), and the real database schema work together correctly, which a unit test with a mocked repository cannot verify in principle.

```ts
describe('OrderRepository (integration)', () => {
  let repository: OrderRepository;
  let dataSource: DataSource;

  beforeAll(async () => {
    // a real test database (e.g., via Testcontainers), not a mock
    dataSource = await createTestDataSource();
    repository = new OrderRepository(dataSource);
  });

  afterAll(() => dataSource.destroy());

  it('saves and retrieves an order', async () => {
    const saved = await repository.save({ amount: 100 });
    const found = await repository.findOne(saved.id);
    expect(found.amount).toBe(100);
  });
});
```

## E2E Tests

Verify the entire application as a black box — a real HTTP request against a running Nest application, with real (or test) databases and external dependencies. The closest thing to how the application will actually be used.

```ts
describe('OrdersController (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({ imports: [AppModule] }).compile();
    app = moduleRef.createNestApplication();
    await app.init();
  });

  afterAll(() => app.close());

  it('POST /orders creates an order', () => {
    return request(app.getHttpServer())
      .post('/orders')
      .send({ amount: 100 })
      .expect(201)
      .expect((res) => {
        expect(res.body.id).toBeDefined();
      });
  });
});
```

## Why the Proportions Matter

- **Unit tests alone** give a false sense of security: each class is verified individually, but there's no guarantee they work correctly together (for example, that an ORM query in the repository actually matches the real database schema).
- **E2E tests alone** are slow (seconds per test instead of milliseconds), brittle (they fail for unrelated reasons — network, data ordering in the database), and poorly localize the cause of failure — one of dozens of steps inside the application failed.

The right proportion is a trade-off between feedback speed (unit) and confidence that the system works as a whole (e2e), with integration tests as the middle layer for the riskiest seams (database, external APIs).

<!-- Open for expansion: Testcontainers for an isolated test database in CI, contract tests for external APIs, when writing more e2e tests is justified -->
