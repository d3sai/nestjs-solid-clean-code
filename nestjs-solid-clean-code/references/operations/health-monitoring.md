---
note: Section added beyond the originally agreed-upon scope — enterprise/senior-level production readiness, at the user's request.
---

# Health Checks and Monitoring

## Core Idea

Without a way to check the application's state, it's hard to tell why it's crashing in a Kubernetes or Docker cluster. NestJS uses `@nestjs/terminus` for Health Checks, and integration with Prometheus/Grafana for metrics.

## Health Checks via Terminus

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

Kubernetes (or any orchestrator) regularly polls `/health` and, based on the response, decides whether to consider the pod "alive" (liveness) and "ready to receive traffic" (readiness).

## Monitoring via Prometheus/Grafana

Prometheus collects application metrics (request count, response time, errors), and Grafana visualizes them in dashboards. This lets you spot a performance degradation before it becomes a critical incident, rather than finding out about a problem after the fact, from user complaints.

## Liveness vs. Readiness

The orchestrator (Kubernetes) checks two different questions, which is why there are often two separate endpoints:

- **Liveness** ("is the application alive?") — if not, the orchestrator restarts the pod. It must not depend on external services (DB, Redis) — otherwise a temporary database outage would trigger cascading restarts of an application that is otherwise healthy.
- **Readiness** ("is the application ready to receive traffic?") — if not, the orchestrator temporarily stops routing requests to the pod, but does NOT restart it. This is the right place to check the database, the queue, external dependencies.

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
    return this.health.check([]); // just returning 200 means "the process is alive"
  }

  @Get('ready')
  @HealthCheck()
  readiness() {
    return this.health.check([() => this.db.pingCheck('database')]);
  }
}
```

## Graceful Shutdown

When the orchestrator stops a pod (deploying a new version, scaling down), the application receives a `SIGTERM` signal and has time to shut down cleanly — instead of dropping requests that are in flight at that moment.

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.enableShutdownHooks(); // Nest will call onModuleDestroy() on providers when SIGTERM is received
  await app.listen(3000);
}
```

```ts
@Injectable()
export class OrderQueueConsumer implements OnModuleDestroy {
  async onModuleDestroy() {
    // wait for jobs already in progress to finish before the process exits
    await this.worker.close();
  }
}
```

Without graceful shutdown, requests that were being processed at the moment of `SIGTERM` (for example, a database write or a call to a payment provider) can be cut off mid-flight — which is especially dangerous for operations that aren't idempotent (`references/operations/idempotency.md`).

## Why This Matters

Health Checks and Monitoring aren't about "clean code" in the SOLID sense — they're about operational reliability: without them, the team finds out about a production outage much later and with far less context for diagnosing it. Graceful shutdown and a proper split between liveness and readiness directly affect how many users notice a problem during a routine deployment of a new version.

<!-- Open for expansion: a detailed example of Prometheus metrics via @willsoto/nestjs-prometheus, configuring the SIGTERM timeout in Kubernetes (terminationGracePeriodSeconds) -->
