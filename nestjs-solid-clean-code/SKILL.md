---
name: nestjs-solid-clean-code
description: The right way to write NestJS & TypeScript clean code — SOLID principles (SRP, OCP, LSP, ISP, DIP) plus DRY/KISS/YAGNI, layered architecture, dependency injection, DTOs & validation, testing, security, logging, caching, and production-readiness practices. Use this skill whenever writing, reviewing, or refactoring any NestJS service, controller, module, repository, guard, or architecture — even if the user doesn't say "SOLID" or "clean code" explicitly. Trigger on requests like "review this service", "how should I structure this module", "is this good NestJS code", "why is this hard to test", or any NestJS/TypeScript architecture, dependency-injection, or code-quality question.
---

# NestJS SOLID & Clean Code

A practical guide to writing NestJS/TypeScript code that you're not ashamed to share and that's easy to maintain for years. At its core are the five SOLID principles, and built around them is everything that, in practice, makes a NestJS application truly production-ready: architecture, DI, security, testing, observability.

## How to use this skill

Don't read all the reference files at once — this `SKILL.md` is just a map. Always keep the 5 SOLID principles (`references/solid/`) in mind as the baseline filter for any NestJS code, then read the 1-3 specific reference files relevant to the task, using the tables below.

When writing, reviewing, or refactoring NestJS code:
1. First check the code through the SOLID lens — does it violate SRP/OCP/LSP/ISP/DIP.
2. Next check the adjacent principles — DRY/KISS/YAGNI/Composition (`references/principles/`) — is the code overengineered or prematurely abstracted.
3. Then — the specific topic of the task (architecture, DI, security, testing, etc.) using the "Quick Guide" table below.

## Contents

### 1. SOLID — fundamental principles (`references/solid/`)

| File | What it covers |
|---|---|
| `srp.md` | Single Responsibility — one class, one reason to change |
| `ocp.md` | Open-Closed — extension through new classes (Strategy), not edits to existing ones |
| `lsp.md` | Liskov Substitution — `implements` instead of `extends`, interchangeability of implementations |
| `isp.md` | Interface Segregation — narrow interfaces instead of "fat" ones |
| `dip.md` | Dependency Inversion — depending on abstractions, DI tokens |

### 2. Adjacent clean-code principles (`references/principles/`)

| File | What it covers |
|---|---|
| `dry.md` | Don't Repeat Yourself, and the risk of premature abstraction |
| `kiss.md` | Keep It Simple — guard clauses, the "5-minute test" |
| `yagni.md` | Not building flexibility "just in case" |
| `composition-over-inheritance.md` | Composing services through DI instead of deep inheritance |

### 3. Architecture and project structure (`references/architecture/`)

| File | What it covers |
|---|---|
| `layered-architecture.md` | Controller → Service → Repository, layer boundaries |
| `module-boundaries.md` | Bounded Contexts, DDD Lite, communication between modules |
| `naming-conventions.md` | Feature-based folder structure, `common`/`shared` |
| `configuration.md` | `ConfigService`, `.env`, environment configuration |
| `cqrs.md` | Command/Query Responsibility Segregation, when it's justified |
| `event-driven-architecture.md` | `EventEmitterModule`, decoupling through events |
| `transactions.md` | Database Transactions / Unit of Work |

### 4. Dependency Injection (`references/dependency-injection/`)

| File | What it covers |
|---|---|
| `di-basics.md` | `@Injectable()`, providers, tokens, `@Inject()` |
| `di-anti-patterns.md` | Service Locator, Circular Deps, Constructor Bloat, Fat Objects |
| `request-scoping.md` | Singleton vs REQUEST vs TRANSIENT, impact on performance |

### 5. Data and validation (`references/data-validation/`)

| File | What it covers |
|---|---|
| `dto-validation.md` | DTO vs Entity, `class-validator`, `ValidationPipe` |
| `repository-pattern.md` | Abstraction over the ORM, and when the Repository Pattern is unnecessary overhead |

### 6. Cross-cutting logic (`references/cross-cutting/`)

| File | What it covers |
|---|---|
| `exception-filters.md` | Global error handling instead of `try-catch` in every method |
| `interceptors-pipes-guards.md` | Guards / Pipes / Interceptors, execution order in the request lifecycle |

### 7. Security (`references/security/`)

| File | What it covers |
|---|---|
| `authentication.md` | JWT, Passport strategies, password hashing, access/refresh tokens |
| `authorization.md` | RBAC via Guards, ABAC/policy-based (CASL) |
| `security-hardening.md` | Helmet, CORS, rate limiting, secrets, protection against injections |

### 8. Production Operations (`references/operations/`)

| File | What it covers |
|---|---|
| `logging.md` | Structured logging, correlation ID, what must not be logged |
| `caching.md` | `@nestjs/cache-manager`, Redis, invalidation strategies |
| `background-jobs.md` | BullMQ, queued jobs instead of long synchronous operations |
| `db-migrations.md` | Prisma Migrate / TypeORM migrations, seeding, why not `synchronize: true` |
| `idempotency.md` | Idempotency Key for retries and webhooks (especially payments) |
| `health-monitoring.md` | Terminus health checks, liveness/readiness, graceful shutdown, Prometheus |
| `api-versioning.md` | API versioning (`/v1/`, `/v2/`) without breaking clients |
| `api-documentation.md` | ⚠️ Currently a stub (commented out) — Swagger/OpenAPI not yet filled in |

### 9. Testing (`references/testing/`)

| File | What it covers |
|---|---|
| `testing-with-solid.md` | Unit tests via mock providers, the connection to DIP/SRP/ISP |
| `testing-pyramid.md` | Unit vs Integration vs E2E, which level to write more of and when |

### 10. Tooling (`references/tooling/`)

| File | What it covers |
|---|---|
| `code-quality-tooling.md` | ESLint, Prettier, Husky/lint-staged, Conventional Commits |

## Quick guide: what to read depending on the task

| Task | Read |
|---|---|
| Writing a new service/controller from scratch | `solid/srp.md`, `dependency-injection/di-basics.md`, `data-validation/dto-validation.md` |
| Reviewing someone else's NestJS code / PR | `solid/*` (all 5), `principles/*`, `dependency-injection/di-anti-patterns.md` |
| Designing a new module/feature | `architecture/layered-architecture.md`, `architecture/module-boundaries.md`, `architecture/naming-conventions.md` |
| A class is hard to test | `dependency-injection/di-anti-patterns.md`, `testing/testing-with-solid.md`, `solid/dip.md` |
| Endpoint access/security questions | `security/authentication.md`, `security/authorization.md`, `cross-cutting/interceptors-pipes-guards.md` |
| Preparing the application for production | `operations/health-monitoring.md`, `operations/logging.md`, `security/security-hardening.md`, `operations/db-migrations.md` |
| Payments / webhook / retry operations | `operations/idempotency.md`, `architecture/transactions.md` |
| A class/service seems overengineered | `principles/kiss.md`, `principles/yagni.md`, `data-validation/repository-pattern.md` (the "when it's unnecessary overhead" section) |
| Writing tests | `testing/testing-with-solid.md`, `testing/testing-pyramid.md` |

Each reference file follows this structure: **the essence of the principle → a "❌ bad / ✅ good" code example → why it matters → cross-references to related topics**. Some files end with a `<!-- Open for expansion -->` comment — these are open points for further expansion of the skill.
