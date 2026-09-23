# Міграції бази даних

## Суть підходу

Зміни схеми БД (нова таблиця, нова колонка, зміна типу) повинні бути версійованими, відтворюваними й однаковими на всіх середовищах (dev, staging, prod) — так само, як версіонується код. Міграція — це файл, що описує конкретну зміну схеми і як її відкотити.

## Чому не можна покладатись на автосинхронізацію в проді

TypeORM має опцію `synchronize: true`, що автоматично підганяє схему БД під поточні Entity під час старту застосунку. Це зручно для локальної розробки, але вкрай небезпечно для проду:

```ts
// ❌ Небезпечно в проді: може видалити колонку чи таблицю без попередження,
// якщо Entity в коді розійшлася зі схемою БД
TypeOrmModule.forRoot({ synchronize: true });
```

```ts
// ✅ Безпечно: схема змінюється лише через явні, переглянуті міграції
TypeOrmModule.forRoot({ synchronize: false });
```

## Приклад з TypeORM

```bash
# генерація міграції на основі різниці між Entity та поточною схемою
npx typeorm migration:generate src/migrations/AddOrderStatus -d src/data-source.ts

# застосування міграцій на цільовій БД
npx typeorm migration:run -d src/data-source.ts
```

```ts
export class AddOrderStatus1234567890 implements MigrationInterface {
  async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`ALTER TABLE "order" ADD "status" varchar NOT NULL DEFAULT 'pending'`);
  }

  async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`ALTER TABLE "order" DROP COLUMN "status"`);
  }
}
```

## Приклад з Prisma Migrate

```bash
# створює міграцію на основі змін у schema.prisma та одразу застосовує локально
npx prisma migrate dev --name add_order_status

# застосування вже готових міграцій на staging/prod (без інтерактивності)
npx prisma migrate deploy
```

Prisma автоматично генерує SQL-файл міграції в `prisma/migrations/`, який комітиться в репозиторій разом з рештою коду.

## Seeding: відтворювані тестові дані

Для dev/staging середовищ корисно мати скрипт, що наповнює БД передбачуваними даними (наприклад, тестового адміна, набір демо-товарів), щоб кожен розробник і CI отримували однаковий стартовий стан:

```ts
// prisma/seed.ts
async function main() {
  await prisma.user.create({
    data: { email: 'admin@example.com', passwordHash: await bcrypt.hash('admin123', 10), role: 'admin' },
  });
}
```

## Чому це важливо

- **Відтворюваність.** Будь-яке середовище (нове оточення розробника, staging, prod) можна привести до однакового стану БД, застосувавши ту саму послідовність міграцій.
- **Ревʼю змін схеми.** Міграція — це файл у Pull Request, який команда може переглянути й обговорити до застосування, на відміну від "магічної" автосинхронізації.
- **Відкат.** У TypeORM-міграціях метод `down()` дозволяє відкотити зміну, якщо після деплою щось пішло не так.

<!-- Місце для доповнення: zero-downtime міграції для великих таблиць (додавання NOT NULL колонки без блокування), стратегія міграцій у CI/CD pipeline -->
