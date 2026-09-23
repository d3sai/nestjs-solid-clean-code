# Фонові задачі (Background Jobs)

## Суть підходу

Не кожна операція повинна виконуватись синхронно всередині HTTP-запиту. Довгі чи ресурсомісткі задачі (відправка email, генерація звіту, обробка зображення) варто виносити в чергу — щоб клієнт отримав відповідь одразу, а сама робота виконалась асинхронно, у фоні.

## Проблема без черги

```ts
// ❌ Погано: клієнт чекає на завершення довгої операції (генерація PDF-звіту)
@Post('reports')
async generateReport(@Body() dto: ReportDto) {
  const report = await this.reportService.generate(dto); // може тривати хвилини
  await this.emailService.sendReport(report);
  return { status: 'sent' };
}
```

Якщо генерація звіту триває довго, HTTP-з'єднання може обірватись за таймаутом, а користувач не отримає жодної відповіді — попри те, що робота, можливо, все ж виконалась.

## Рішення: BullMQ (`@nestjs/bullmq`)

```ts
@Module({
  imports: [
    BullModule.forRoot({ connection: { host: 'localhost', port: 6379 } }),
    BullModule.registerQueue({ name: 'reports' }),
  ],
})
export class ReportsModule {}
```

```ts
@Injectable()
export class ReportsService {
  constructor(@InjectQueue('reports') private readonly reportsQueue: Queue) {}

  async requestReport(dto: ReportDto) {
    await this.reportsQueue.add('generate', dto);
    return { status: 'queued' }; // відповідь одразу, без очікування
  }
}
```

```ts
@Processor('reports')
export class ReportsProcessor extends WorkerHost {
  constructor(private readonly emailService: EmailService) {
    super();
  }

  async process(job: Job<ReportDto>) {
    const report = await this.generate(job.data);
    await this.emailService.sendReport(report);
  }

  private async generate(dto: ReportDto) {
    // важка робота виконується тут, окремо від HTTP-запиту
  }
}
```

Ендпоінт одразу повертає `{ status: 'queued' }`, а сама генерація звіту й відправка листа відбуваються в окремому процесі-воркері.

## Чому це важливо

- **Стійкість до збоїв.** Черга зберігає задачу навіть якщо застосунок перезапуститься — на відміну від `setTimeout`/`Promise`, що виконуються "в пам'яті" й губляться при падінні процесу.
- **Retry з backoff.** BullMQ з коробки підтримує повторні спроби при невдачі (наприклад, тимчасова недоступність SMTP-сервера) без ручного написання цієї логіки в кожному сервісі.
- **Контроль навантаження.** Кількість одночасних воркерів (`concurrency`) обмежує навантаження на зовнішні сервіси (наприклад, API платіжної системи з лімітом запитів).

## Коли черга не потрібна

Для швидких операцій (десятки-сотні мілісекунд), де відповідь потрібна одразу і синхронно (наприклад, підтвердження оплати перед показом результату користувачу), черга лише додає складності без вигоди — знову ж таки, оцінюйте через YAGNI (`references/principles/yagni.md`).

<!-- Місце для доповнення: пріоритети задач, dead-letter queue для задач, що постійно падають, моніторинг довжини черги (Bull Board) -->
