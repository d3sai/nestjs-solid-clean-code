# Database Migrations

## Core Idea

Changes to the database schema (a new table, a new column, a type change) must be versioned, reproducible, and identical across all environments (dev, staging, prod) — just like code is versioned. A migration is a file that describes a specific schema change and how to roll it back.

## Why You Can't Rely on Auto-Sync in Production

TypeORM has a `synchronize: true` option that automatically adjusts the database schema to match the current Entities on application startup. This is convenient for local development, but extremely dangerous in production:

```ts
// ❌ Dangerous in production: can drop a column or table without warning
// if the Entity in code has drifted from the database schema
TypeOrmModule.forRoot({ synchronize: true });
```

```ts
// ✅ Safe: the schema only changes through explicit, reviewed migrations
TypeOrmModule.forRoot({ synchronize: false });
```

## Example with TypeORM

```bash
# generate a migration based on the diff between Entities and the current schema
npx typeorm migration:generate src/migrations/AddOrderStatus -d src/data-source.ts

# apply migrations to the target database
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

## Example with Prisma Migrate

```bash
# creates a migration based on changes to schema.prisma and applies it locally right away
npx prisma migrate dev --name add_order_status

# applies already-generated migrations to staging/prod (non-interactive)
npx prisma migrate deploy
```

Prisma automatically generates a SQL migration file under `prisma/migrations/`, which is committed to the repository along with the rest of the code.

## Seeding: Reproducible Test Data

For dev/staging environments, it's useful to have a script that populates the database with predictable data (for example, a test admin account, a set of demo products), so every developer and CI run start from the same initial state:

```ts
// prisma/seed.ts
async function main() {
  await prisma.user.create({
    data: { email: 'admin@example.com', passwordHash: await bcrypt.hash('admin123', 10), role: 'admin' },
  });
}
```

## Why This Matters

- **Reproducibility.** Any environment (a new developer's setup, staging, prod) can be brought to the same database state by applying the same sequence of migrations.
- **Reviewing schema changes.** A migration is a file in a Pull Request that the team can review and discuss before it's applied, unlike "magic" auto-sync.
- **Rollback.** In TypeORM migrations, the `down()` method lets you roll back a change if something goes wrong after deployment.

<!-- Open for expansion: zero-downtime migrations for large tables (adding a NOT NULL column without locking), migration strategy in a CI/CD pipeline -->
