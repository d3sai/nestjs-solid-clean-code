# Repository Pattern

## Core Idea

The Repository Pattern separates business logic from direct database access. It makes code cleaner, makes testing easier, and is a direct implementation of the **Dependency Inversion Principle** — the service depends on a data-access abstraction, not on a concrete ORM.

## How to Implement the Repository Pattern

### 1. Repository Interface

An abstraction that describes the methods for working with data (`findAll`, `findOne`, `create`, etc.). The service that depends on this interface doesn't know which ORM is actually being used under the hood.

```ts
export interface UsersRepository {
  findAll(): Promise<User[]>;
  findOne(id: number): Promise<User | null>;
  create(data: CreateUserDto): Promise<User>;
}

export const USERS_REPOSITORY = 'USERS_REPOSITORY';
```

### 2. Concrete Implementation

A class that implements the interface and calls the methods of a specific database library:

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

### 3. Registration in the Module (Dependency Injection)

The interface is bound to a concrete implementation via `providers`:

```ts
@Module({
  providers: [
    { provide: USERS_REPOSITORY, useClass: PrismaUsersRepository },
  ],
  exports: [USERS_REPOSITORY],
})
export class UsersModule {}
```

### 4. Usage in the Service

The repository is injected into the service's constructor via `@Inject()` with the interface's token:

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

`UsersService` only knows about the `UsersRepository` contract — it doesn't care whether it's Prisma, TypeORM, or something else.

## Why It Pays Off

- **Testing.** The real repository is easy to swap for a mock in unit tests (`useValue` with `jest.fn()` methods) — no real database needed. See `references/testing/testing-with-solid.md` for details.
- **Flexibility.** If you decide to change the database or ORM library (e.g., Prisma → TypeORM), you only need to change one implementation class (`PrismaUsersRepository` → `TypeOrmUsersRepository`) — no application service needs to be touched.
- **Cleanliness.** The service handles only business logic, not building database queries — this aligns with `references/architecture/layered-architecture.md`.

## When the Repository Pattern Is Unnecessary Overhead

The Repository Pattern is a powerful tool, but it isn't always necessary or efficient. Sometimes a simpler approach is the better solution (this is the same **YAGNI**, see `references/principles/yagni.md`).

### When to Skip or Simplify the Approach

1. **Small projects (CRUD apps).** If the application is essentially a simple "interface to a database" with minimal business logic, a separate repository layer adds nothing but boilerplate: extra interfaces and classes that slow down development without providing real value.

2. **Feature-rich ORMs.** Modern ORMs (Prisma, TypeORM, Entity Framework) already implement the Repository and Unit of Work patterns internally. If `UsersRepository` is just a wrapper around `this.prisma.user.findMany()` with no additional logic, it's an unnecessary abstraction that only makes the code harder to navigate:

   ```ts
   // ❌ Abstraction for abstraction's sake — no real value
   @Injectable()
   export class UsersRepository {
     constructor(private readonly prisma: PrismaService) {}

     findAll() {
       return this.prisma.user.findMany(); // 1-to-1 pass-through
     }
   }
   ```

3. **Limited time (MVP).** At the prototype stage, speed to market matters more than architectural purity. A complex pattern becomes overhead that eats into time better spent building the product.

4. **Very complex, specific queries.** If database queries are so complex that the repository turns into a "pass-through" layer (ORM parameters still flow through every layer unchanged), the abstraction loses its point — using ORM methods directly in the service layer will be more readable.

### When the Repository Pattern Is Definitely Needed

- When the data source is expected to change frequently (e.g., moving from SQL to NoSQL).
- When you need to test business logic without connecting to a real database.
- In large enterprise systems with a clear domain split (DDD), where it's essential for maintaining code across multiple teams.

<!-- Open for expansion: a generic repository (BaseRepository<T>), the difference between the Repository Pattern and the built-in TypeORM Repository -->
