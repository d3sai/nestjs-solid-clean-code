# Background Jobs

## Core Idea

Not every operation has to run synchronously inside an HTTP request. Long-running or resource-intensive tasks (sending email, generating a report, processing an image) should be offloaded to a queue — so the client gets a response immediately, while the actual work runs asynchronously, in the background.

## The Problem Without a Queue

```ts
// ❌ Bad: the client waits for a long-running operation to finish (generating a PDF report)
@Post('reports')
async generateReport(@Body() dto: ReportDto) {
  const report = await this.reportService.generate(dto); // can take minutes
  await this.emailService.sendReport(report);
  return { status: 'sent' };
}
```

If report generation takes a long time, the HTTP connection may time out, and the user gets no response at all — even though the work may still have completed successfully.

## Solution: BullMQ (`@nestjs/bullmq`)

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
    return { status: 'queued' }; // respond immediately, without waiting
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
    // the heavy work happens here, separate from the HTTP request
  }
}
```

The endpoint immediately returns `{ status: 'queued' }`, while the actual report generation and email sending happen in a separate worker process.

## Why This Matters

- **Resilience to failures.** A queue persists a task even if the application restarts — unlike `setTimeout`/`Promise`, which run "in memory" and are lost if the process crashes.
- **Retry with backoff.** BullMQ supports retries on failure out of the box (for example, a temporary SMTP server outage) without hand-writing that logic in every service.
- **Load control.** The number of concurrent workers (`concurrency`) caps the load on external services (for example, a payment provider's API with a rate limit).

## When You Don't Need a Queue

For fast operations (tens to hundreds of milliseconds) where the response is needed immediately and synchronously (for example, confirming a payment before showing the result to the user), a queue only adds complexity without benefit — again, evaluate this through YAGNI (`references/principles/yagni.md`).

<!-- Open for expansion: job priorities, a dead-letter queue for jobs that keep failing, monitoring queue length (Bull Board) -->
