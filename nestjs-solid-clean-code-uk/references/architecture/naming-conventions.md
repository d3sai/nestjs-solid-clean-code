# Структура проєкту та конвенції найменування

## Суть підходу

Правильна структура NestJS-проєкту базується на модульній архітектурі, де кожна функціональна частина програми (фіча) ізольована в окремій папці. Це робить код передбачуваним, легким для тестування та масштабування.

## Рекомендована структура (Feature-based)

```
src/
├── main.ts                  # точка входу в застосунок
├── app.module.ts             # кореневий модуль, що об'єднує всі інші
├── modules/                  # бізнес-модулі
│   ├── users/
│   │   ├── dto/               # Data Transfer Objects для валідації запитів
│   │   ├── entities/          # класи моделей даних (сутності БД)
│   │   ├── controllers/       # обробка HTTP-запитів
│   │   ├── services/          # бізнес-логіка
│   │   └── users.module.ts    # реєстрація всіх компонентів модуля
│   ├── orders/
│   │   └── ...
│   └── auth/
│       └── ...
├── common/                    # загальні компоненти для всього застосунку
│   ├── filters/                # exception filters
│   ├── interceptors/
│   ├── decorators/
│   └── guards/
└── config/                    # налаштування середовища, конфігураційні файли
```

## Ключові поради для Senior-рівня

### 1. Модульність (Feature-based, а не Layer-based)

Замість того, щоб групувати файли за типом (всі контролери в одній папці `controllers/`, всі сервіси — в іншій `services/`), групуйте їх за функціональністю (фічею): все, що стосується `users`, лежить в `modules/users/`.

```
// ❌ Layer-based: логіка однієї фічі розкидана по всьому проєкту
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
// ✅ Feature-based: все про users в одному місці
src/modules/users/
├── dto/create-user.dto.ts
├── entities/user.entity.ts
├── controllers/users.controller.ts
├── services/users.service.ts
└── users.module.ts
```

Це дозволяє легко видалити або перенести весь модуль (наприклад, виділити його в окремий мікросервіс пізніше), не ламаючи структуру решти проєкту.

### 2. Багаторівневість (Repository Pattern у складних проєктах)

Якщо проєкт складний, сервіс не повинен напряму звертатися до ORM (наприклад, Prisma чи TypeORM) — взаємодія має йти через репозиторій. Детальніше — у `references/data-validation/repository-pattern.md` та `references/architecture/layered-architecture.md`.

### 3. DTO та Validation

Завжди виділяйте DTO окремо від Entity — це захищає код від прямого доступу до внутрішньої структури об'єктів бази даних. Детальніше — у `references/data-validation/dto-validation.md`.

### 4. Глобальний контекст (папка `common`/`shared`)

Речі, що використовуються в усьому проєкті (логування, обробка помилок, спільні декоратори та guard-и), виносяться в окрему папку `common` (або `shared`), а не дублюються чи розмазуються по фіча-модулях.

<!-- Місце для доповнення: barrel-файли (index.ts) та їхні недоліки, конвенції іменування файлів (kebab-case, суфікси .dto/.entity/.service), core-модуль для синглтонів -->
