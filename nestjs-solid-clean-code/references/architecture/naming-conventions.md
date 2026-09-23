# Project Structure and Naming Conventions

## Core Idea

A well-structured NestJS project is built on modular architecture, where each functional part of the application (a feature) is isolated in its own folder. This makes the code predictable, easy to test, and easy to scale.

## Recommended Structure (Feature-based)

```
src/
├── main.ts                  # application entry point
├── app.module.ts             # root module that ties everything together
├── modules/                  # business modules
│   ├── users/
│   │   ├── dto/               # Data Transfer Objects for request validation
│   │   ├── entities/          # data model classes (database entities)
│   │   ├── controllers/       # HTTP request handling
│   │   ├── services/          # business logic
│   │   └── users.module.ts    # registration of all module components
│   ├── orders/
│   │   └── ...
│   └── auth/
│       └── ...
├── common/                    # shared components used across the whole app
│   ├── filters/                # exception filters
│   ├── interceptors/
│   ├── decorators/
│   └── guards/
└── config/                    # environment settings, configuration files
```

## Key Tips for the Senior Level

### 1. Modularity (Feature-based, not Layer-based)

Instead of grouping files by type (all controllers in one `controllers/` folder, all services in another `services/` folder), group them by functionality (feature): everything related to `users` lives in `modules/users/`.

```
// ❌ Layer-based: the logic of a single feature is scattered across the whole project
src/
├── controllers/
│   ├── users.controller.ts
│   └── orders.controller.ts
├── services/
│   ├── users.service.ts
│   └── orders.service.ts
└── dto/
    ├── create-user.dto.ts
    └── create-order.dto.ts
```

```
// ✅ Feature-based: everything about users lives in one place
src/modules/users/
├── dto/create-user.dto.ts
├── entities/user.entity.ts
├── controllers/users.controller.ts
├── services/users.service.ts
└── users.module.ts
```

This makes it easy to delete or move an entire module (for example, extracting it into a separate microservice later) without breaking the structure of the rest of the project.

### 2. Layering (Repository Pattern in Complex Projects)

If the project is complex, a service should not talk to the ORM (for example, Prisma or TypeORM) directly — the interaction should go through a repository. For more detail, see `references/data-validation/repository-pattern.md` and `references/architecture/layered-architecture.md`.

### 3. DTOs and Validation

Always keep DTOs separate from Entities — this protects the code from direct access to the internal structure of database objects. For more detail, see `references/data-validation/dto-validation.md`.

### 4. Global Context (the `common`/`shared` folder)

Things used across the entire project (logging, error handling, shared decorators and guards) are moved into a separate `common` (or `shared`) folder, rather than duplicated or scattered across feature modules.

<!-- Open for expansion: barrel files (index.ts) and their downsides, file naming conventions (kebab-case, .dto/.entity/.service suffixes), a core module for singletons -->
