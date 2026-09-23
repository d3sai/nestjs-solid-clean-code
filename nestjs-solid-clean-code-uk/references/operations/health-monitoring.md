---
note: Розділ додано понад початково узгоджений зміст — Enterprise/Senior-рівень production-readiness, за запитом користувача.
---

# Health Checks та Monitoring

## Суть підходу

Без перевірки стану застосунку складно зрозуміти, чому він падає в Kubernetes чи Docker-кластері. Для цього в NestJS використовується `@nestjs/terminus` для Health Checks, а для метрик — інтеграція з Prometheus/Grafana.

## Health Checks через Terminus

```ts
@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly db: TypeOrmHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('database'),
    ]);
  }
}
```

Kubernetes (чи будь-який оркестратор) регулярно звертається до `/health` і на основі відповіді вирішує, чи вважати под "живим" (liveness) і "готовим приймати трафік" (readiness).

## Monitoring через Prometheus/Grafana

Prometheus збирає метрики застосунку (кількість запитів, час відповіді, помилки), а Grafana візуалізує їх у дашборди. Це дозволяє побачити деградацію продуктивності до того, як вона стане критичним інцидентом, а не дізнаватись про проблему постфактум зі скарг користувачів.

## Liveness vs Readiness

Оркестратор (Kubernetes) перевіряє два різні питання, тому часто мають бути два окремі ендпоінти:

- **Liveness** ("застосунок живий?") — якщо ні, оркестратор перезапускає под. Не повинен залежати від зовнішніх сервісів (БД, Redis) — інакше тимчасова недоступність БД спричинить каскадні перезапуски застосунку, який сам по собі здоровий.
- **Readiness** ("застосунок готовий приймати трафік?") — якщо ні, оркестратор тимчасово перестає направляти на под запити, але НЕ перезапускає його. Тут якраз доречно перевіряти БД, чергу, зовнішні залежності.

```ts
@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly db: TypeOrmHealthIndicator,
  ) {}

  @Get('live')
  @HealthCheck()
  liveness() {
    return this.health.check([]); // сам факт відповіді 200 означає "процес живий"
  }

  @Get('ready')
  @HealthCheck()
  readiness() {
    return this.health.check([() => this.db.pingCheck('database')]);
  }
}
```

## Graceful Shutdown

Коли оркестратор зупиняє под (деплой нової версії, масштабування), застосунок отримує сигнал `SIGTERM` і має час коректно завершити роботу — а не обірвати запити, що виконуються в цей момент.

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableShutdownHooks(); // Nest сам викличе onModuleDestroy() у провайдерах при SIGTERM
  await app.listen(3000);
}
```

```ts
@Injectable()
export class OrderQueueConsumer implements OnModuleDestroy {
  async onModuleDestroy() {
    // дочекатись завершення задач, що вже виконуються, перш ніж процес завершиться
    await this.worker.close();
  }
}
```

Без graceful shutdown запити, що були в процесі виконання в момент `SIGTERM` (наприклад, запис у БД чи виклик платіжного провайдера), можуть обірватись на середині — що особливо небезпечно для операцій без ідемпотентності (`references/operations/idempotency.md`).

## Чому це важливо

Health Checks і Monitoring — не про "чистоту коду" в сенсі SOLID, а про операційну надійність: без них команда дізнається про падіння застосунку в проді значно пізніше і з меншою кількістю контексту для діагностики. Graceful shutdown і правильний поділ liveness/readiness напряму впливають на те, скільки користувачів відчують проблему під час звичайного деплою нової версії.

<!-- Місце для доповнення: детальний приклад Prometheus-метрик через @willsoto/nestjs-prometheus, налаштування timeout для SIGTERM у Kubernetes (terminationGracePeriodSeconds) -->
