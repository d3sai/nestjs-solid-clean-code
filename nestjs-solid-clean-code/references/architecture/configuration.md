---
note: This file was added beyond the originally agreed table of contents — the topic of configuration matters for "senior-level" NestJS code, so it was broken out into a separate reference.
---

# Configuration via ConfigService

## Core Idea

Never hardcode API keys, passwords, or ports directly in the code. Instead, use `@nestjs/config` and `.env` files — this is the standard way of working with different environments (dev, staging, prod).

## The Problem With Hardcoding

```ts
// ❌ Bad: secrets and settings right in the code
@Injectable()
export class MailService {
  private readonly apiKey = 'sk_live_51H8xJ2...'; // will end up in the git repository!
  private readonly port = 3000;
}
```

## Solution: ConfigModule + ConfigService

```env
# .env
DATABASE_URL=postgres://user:pass@localhost:5432/app
MAIL_API_KEY=sk_live_51H8xJ2...
PORT=3000
```

```ts
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: `.env.${process.env.NODE_ENV || 'development'}`,
    }),
  ],
})
export class AppModule {}
```

```ts
@Injectable()
export class MailService {
  constructor(private readonly configService: ConfigService) {}

  send() {
    const apiKey = this.configService.get<string>('MAIL_API_KEY');
    // ...
  }
}
```

## Why This Matters

- **Security:** secrets don't end up in the git repository (`.env` is added to `.gitignore`).
- **Environment flexibility:** dev/staging/prod use different values without any code changes — only different `.env` files or environment variables.
- **DIP at the configuration level:** services depend on the `ConfigService` abstraction, not on where the value actually came from (a file, an environment variable, a secrets manager).

<!-- Open for expansion: validating .env with Joi/class-validator (ConfigModule.forRoot({ validationSchema })), typed configuration via namespaces (registerAs) -->
