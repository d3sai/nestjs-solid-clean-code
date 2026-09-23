---
name: nestjs-solid-clean-code
description: The right way to write NestJS & TypeScript clean code — SOLID principles (SRP, OCP, LSP, ISP, DIP) plus DRY/KISS/YAGNI, layered architecture, dependency injection, DTOs & validation, testing, security, logging, caching, and production-readiness practices. Use this skill whenever writing, reviewing, or refactoring any NestJS service, controller, module, repository, guard, or architecture — even if the user doesn't say "SOLID" or "clean code" explicitly. Trigger on requests like "review this service", "how should I structure this module", "is this good NestJS code", "why is this hard to test", or any NestJS/TypeScript architecture, dependency-injection, or code-quality question.
---

# NestJS SOLID & Clean Code

Практичний посібник із написання NestJS/TypeScript коду, яким не соромно ділитися і який легко підтримувати роками. В основі — п'ять принципів SOLID, а навколо них — усе, що на практиці робить NestJS-застосунок дійсно production-ready: архітектура, DI, безпека, тестування, спостережуваність.

## Як користуватись цим скілом

Не читай усі reference-файли одразу — це `SKILL.md` лише мапа. Завжди тримай в голові 5 принципів SOLID (`references/solid/`) як базовий фільтр для будь-якого NestJS-коду, а тоді читай 1-3 конкретні reference-файли, що стосуються задачі, за таблицями нижче.

Коли пишеш, рев'юїш чи рефакториш NestJS-код:
1. Спочатку перевір код через призму SOLID — чи не порушує він SRP/OCP/LSP/ISP/DIP.
2. Далі перевір суміжні принципи — DRY/KISS/YAGNI/Composition (`references/principles/`) — чи не переускладнений або передчасно абстрагований код.
3. Далі — конкретна тема задачі (архітектура, DI, безпека, тестування тощо) за таблицею "Швидкий гайд" нижче.

## Зміст

### 1. SOLID — фундаментальні принципи (`references/solid/`)

| Файл | Про що |
|---|---|
| `srp.md` | Single Responsibility — один клас, одна причина для зміни |
| `ocp.md` | Open-Closed — розширення через нові класи (Strategy), а не правки існуючих |
| `lsp.md` | Liskov Substitution — `implements` замість `extends`, взаємозамінність реалізацій |
| `isp.md` | Interface Segregation — вузькі інтерфейси замість "товстих" |
| `dip.md` | Dependency Inversion — залежність від абстракцій, DI-токени |

### 2. Суміжні принципи чистого коду (`references/principles/`)

| Файл | Про що |
|---|---|
| `dry.md` | Don't Repeat Yourself, і ризик передчасної абстракції |
| `kiss.md` | Keep It Simple — guard clauses, "5-хвилинний тест" |
| `yagni.md` | Не будувати гнучкість "про запас" |
| `composition-over-inheritance.md` | Композиція сервісів через DI замість глибокого успадкування |

### 3. Архітектура та структура проєкту (`references/architecture/`)

| Файл | Про що |
|---|---|
| `layered-architecture.md` | Controller → Service → Repository, межі шарів |
| `module-boundaries.md` | Bounded Contexts, DDD Lite, комунікація між модулями |
| `naming-conventions.md` | Feature-based структура папок, `common`/`shared` |
| `configuration.md` | `ConfigService`, `.env`, налаштування середовищ |
| `cqrs.md` | Command/Query Responsibility Segregation, коли виправдано |
| `event-driven-architecture.md` | `EventEmitterModule`, декоплінг через події |
| `transactions.md` | Database Transactions / Unit of Work |

### 4. Dependency Injection (`references/dependency-injection/`)

| Файл | Про що |
|---|---|
| `di-basics.md` | `@Injectable()`, провайдери, токени, `@Inject()` |
| `di-anti-patterns.md` | Service Locator, Circular Deps, Constructor Bloat, Fat Objects |
| `request-scoping.md` | Singleton vs REQUEST vs TRANSIENT, вплив на продуктивність |

### 5. Дані та валідація (`references/data-validation/`)

| Файл | Про що |
|---|---|
| `dto-validation.md` | DTO vs Entity, `class-validator`, `ValidationPipe` |
| `repository-pattern.md` | Абстракція над ORM, і коли Repository Pattern — зайвий overhead |

### 6. Крос-каттінг логіка (`references/cross-cutting/`)

| Файл | Про що |
|---|---|
| `exception-filters.md` | Глобальна обробка помилок замість `try-catch` у кожному методі |
| `interceptors-pipes-guards.md` | Guards / Pipes / Interceptors, порядок виконання в request lifecycle |

### 7. Безпека (`references/security/`)

| Файл | Про що |
|---|---|
| `authentication.md` | JWT, Passport-стратегії, хешування паролів, access/refresh токени |
| `authorization.md` | RBAC через Guards, ABAC/policy-based (CASL) |
| `security-hardening.md` | Helmet, CORS, rate limiting, секрети, захист від ін'єкцій |

### 8. Production Operations (`references/operations/`)

| Файл | Про що |
|---|---|
| `logging.md` | Структуроване логування, correlation ID, що не можна логувати |
| `caching.md` | `@nestjs/cache-manager`, Redis, стратегії інвалідації |
| `background-jobs.md` | BullMQ, чергові задачі замість синхронних довгих операцій |
| `db-migrations.md` | Prisma Migrate / TypeORM migrations, seeding, чому не `synchronize: true` |
| `idempotency.md` | Idempotency Key для ретраїв і webhook-ів (особливо платежі) |
| `health-monitoring.md` | Terminus health checks, liveness/readiness, graceful shutdown, Prometheus |
| `api-versioning.md` | Версіонування API (`/v1/`, `/v2/`) без поломки клієнтів |
| `api-documentation.md` | ⚠️ Наразі заглушка (закоментовано) — Swagger/OpenAPI ще не наповнено |

### 9. Тестування (`references/testing/`)

| Файл | Про що |
|---|---|
| `testing-with-solid.md` | Unit-тести через мок-провайдери, зв'язок з DIP/SRP/ISP |
| `testing-pyramid.md` | Unit vs Integration vs E2E, коли якого рівня писати більше |

### 10. Інструментарій (`references/tooling/`)

| Файл | Про що |
|---|---|
| `code-quality-tooling.md` | ESLint, Prettier, Husky/lint-staged, Conventional Commits |

## Швидкий гайд: що читати залежно від задачі

| Задача | Читай |
|---|---|
| Пишеш новий сервіс/контролер з нуля | `solid/srp.md`, `dependency-injection/di-basics.md`, `data-validation/dto-validation.md` |
| Рев'юїш чужий NestJS-код / PR | `solid/*` (всі 5), `principles/*`, `dependency-injection/di-anti-patterns.md` |
| Проєктуєш новий модуль/фічу | `architecture/layered-architecture.md`, `architecture/module-boundaries.md`, `architecture/naming-conventions.md` |
| Клас важко тестувати | `dependency-injection/di-anti-patterns.md`, `testing/testing-with-solid.md`, `solid/dip.md` |
| Питання доступу/безпеки ендпоінта | `security/authentication.md`, `security/authorization.md`, `cross-cutting/interceptors-pipes-guards.md` |
| Готуєш застосунок до продакшену | `operations/health-monitoring.md`, `operations/logging.md`, `security/security-hardening.md`, `operations/db-migrations.md` |
| Операція з платежами / webhook / ретраї | `operations/idempotency.md`, `architecture/transactions.md` |
| Клас/сервіс здається переускладненим | `principles/kiss.md`, `principles/yagni.md`, `data-validation/repository-pattern.md` (розділ "коли зайвий overhead") |
| Пишеш тести | `testing/testing-with-solid.md`, `testing/testing-pyramid.md` |

Кожен reference-файл має структуру: **суть принципу → приклад коду "❌ погано / ✅ добре" → чому це важливо → перехресні посилання на суміжні теми**. Деякі файли закінчуються коментарем `<!-- Місце для доповнення -->` — це відкриті точки для подальшого розширення скіла.
