# Repository Pattern

## Суть підходу

Repository Pattern відокремлює бізнес-логіку від безпосередньої роботи з базою даних. Це робить код чистішим, полегшує тестування та є прямою реалізацією **Dependency Inversion Principle** — сервіс залежить від абстракції доступу до даних, а не від конкретної ORM.

## Як реалізувати Repository Pattern

### 1. Інтерфейс репозиторію

Абстракція, яка описує методи роботи з даними (`findAll`, `findOne`, `create` тощо). Сервіс, що залежить від цього інтерфейсу, не знає, яка саме ORM використовується під капотом.

```ts
export interface UsersRepository {
  findAll(): Promise<User[]>;
  findOne(id: number): Promise<User | null>;
  create(data: CreateUserDto): Promise<User>;
}

export const USERS_REPOSITORY = 'USERS_REPOSITORY';
```

### 2. Конкретна реалізація

Клас, що реалізує інтерфейс і викликає методи конкретної бібліотеки для роботи з БД:

```ts
@Injectable()
export class PrismaUsersRepository implements UsersRepository {
  constructor(private readonly prisma: PrismaService) {}

  findAll(): Promise<User[]> {
    return this.prisma.user.findMany();
  }

  findOne(id: number): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { id } });
  }

  create(data: CreateUserDto): Promise<User> {
    return this.prisma.user.create({ data });
  }
}
```

### 3. Реєстрація у модулі (Dependency Injection)

Інтерфейс прив'язується до конкретної реалізації через `providers`:

```ts
@Module({
  providers: [
    { provide: USERS_REPOSITORY, useClass: PrismaUsersRepository },
  ],
  exports: [USERS_REPOSITORY],
})
export class UsersModule {}
```

### 4. Використання у сервісі

Репозиторій ін'єктується в конструктор сервісу через `@Inject()` з токеном інтерфейсу:

```ts
@Injectable()
export class UsersService {
  constructor(
    @Inject(USERS_REPOSITORY) private readonly usersRepository: UsersRepository,
  ) {}

  async getAllUsers() {
    return this.usersRepository.findAll();
  }
}
```

`UsersService` знає лише про контракт `UsersRepository` — йому байдуже, Prisma це, TypeORM чи щось інше.

## Чому це вигідно

- **Тестування.** Реальний репозиторій легко підмінити моком під час unit-тестів (`useValue` з jest.fn()-методами) — без реальної БД. Детальніше — у `references/testing/testing-with-solid.md`.
- **Гнучкість.** Якщо ви вирішите змінити базу даних чи бібліотеку ORM (наприклад, Prisma → TypeORM), достатньо змінити лише один клас-реалізацію (`PrismaUsersRepository` → `TypeOrmUsersRepository`) — жоден сервіс застосунку правити не потрібно.
- **Чистота.** Сервіс займається лише бізнес-логікою, а не складанням запитів до БД — це узгоджується з `references/architecture/layered-architecture.md`.

## Коли Repository Pattern — зайвий overhead

Repository Pattern — потужний інструмент, але не завжди необхідний чи ефективний. Іноді простіший підхід — краще рішення (це той самий **YAGNI**, див. `references/principles/yagni.md`).

### Коли варто відмовитись або спростити підхід

1. **Невеликі проєкти (CRUD-додатки).** Якщо застосунок — це, по суті, простий «інтерфейс до бази даних» з мінімальною бізнес-логікою, окремий шар репозиторіїв додає лише boilerplate: зайві інтерфейси й класи, які сповільнюють розробку, не даючи реальної користі.

2. **Потужні ORM.** Сучасні ORM (Prisma, TypeORM, Entity Framework) вже самі по собі реалізують патерни Repository та Unit of Work під капотом. Якщо `UsersRepository` — це просто обгортка над `this.prisma.user.findMany()` без жодної додаткової логіки, це зайва абстракція, яка лише ускладнює навігацію кодом:

   ```ts
   // ❌ Абстракція заради абстракції — жодної реальної цінності
   @Injectable()
   export class UsersRepository {
     constructor(private readonly prisma: PrismaService) {}

     findAll() {
       return this.prisma.user.findMany(); // 1-в-1 прокидання виклику далі
     }
   }
   ```

3. **Обмежені часові ресурси (MVP).** На етапі прототипу швидкість виходу на ринок критичніша за архітектурну чистоту. Складний патерн стає overhead, що забирає час на підтримку замість розвитку продукту.

4. **Дуже складні, специфічні запити.** Якщо запити до БД настільки складні, що репозиторій перетворюється на «транзитивний» шар (параметри ORM все одно прокидаються через усі шари без змін), абстракція втрачає сенс — пряме використання методів ORM у сервісному шарі буде читабельнішим.

### Коли Repository Pattern точно потрібен

- Коли очікується часта зміна джерела даних (наприклад, перехід з SQL на NoSQL).
- Коли потрібно тестувати бізнес-логіку без підключення реальної БД.
- У великих Enterprise-системах з чітким поділом на домени (DDD), де це обов'язково для підтримки коду багатьма командами.

<!-- Місце для доповнення: generic-репозиторій (BaseRepository<T>), різниця між Repository Pattern і вбудованим TypeORM Repository -->
