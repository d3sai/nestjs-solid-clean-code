# Testing Pyramid: Unit, Integration, E2E

## Суть підходу

Не всі тести однаково дешеві й швидкі. Testing Pyramid — модель, що показує, у якій пропорції варто писати тести різних рівнів: багато швидких unit-тестів в основі, менше інтеграційних, і зовсім небагато повільних e2e — на вершині.

```
        /\
       /e2e\        мало, повільні, дорогі в підтримці
      /------\
     /integr. \     помірно, середня швидкість
    /----------\
   /  unit tests \  багато, швидкі, дешеві
  /----------------\
```

## Unit-тести

Перевіряють одну одиницю логіки (сервіс, функцію) повністю ізольовано, з усіма залежностями замоканими. Детальний приклад — `references/testing/testing-with-solid.md`. Швидкі (мілісекунди), тому їх мають бути сотні — вони ловлять регресії в бізнес-логіці на найдешевшому рівні.

## Integration-тести

Перевіряють взаємодію кількох реальних компонентів разом — наприклад, сервіс і реальну (тестову) базу даних, без моків репозиторію. Мета — переконатись, що ORM-запити, транзакції (`references/architecture/transactions.md`) і реальна схема БД працюють разом коректно, чого unit-тест з моком репозиторію в принципі перевірити не може.

```ts
describe('OrderRepository (integration)', () => {
  let repository: OrderRepository;
  let dataSource: DataSource;

  beforeAll(async () => {
    // реальна тестова БД (наприклад, через Testcontainers), не мок
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

## E2E-тести

Перевіряють увесь застосунок як чорну скриньку — реальний HTTP-запит через піднятий Nest-застосунок, з реальними (чи тестовими) БД і зовнішніми залежностями. Найближче до того, як застосунок насправді використовуватиметься.

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

## Чому пропорції важливі

- **Лише unit-тести** дають хибне відчуття безпеки: кожен клас перевірений окремо, але немає гарантії, що вони коректно працюють разом (наприклад, що ORM-запит у репозиторії насправді відповідає реальній схемі БД).
- **Лише e2e-тести** повільні (секунди на тест замість мілісекунд), крихкі (падають від непов'язаних причин — мережа, порядок даних у БД) і погано локалізують причину падіння — впав один з десятків кроків усередині застосунку.

Правильна пропорція — це компроміс між швидкістю зворотного зв'язку (unit) і впевненістю, що система працює як єдине ціле (e2e), з інтеграційними тестами як проміжною ланкою для найризикованіших стиків (БД, зовнішні API).

<!-- Місце для доповнення: Testcontainers для ізольованої тестової БД в CI, contract-тести для зовнішніх API, коли e2e виправдано писати більше -->
