---
note: Цей файл додано понад початково узгоджений зміст — тема конфігурації важлива для "senior-рівня" NestJS коду, тож винесена в окремий reference.
---

# Конфігурація через ConfigService

## Суть підходу

Ніколи не хардкодьте ключі API, паролі чи порти прямо в коді. Замість цього використовується `@nestjs/config` та файли `.env` — це стандарт для роботи з різними середовищами (dev, staging, prod).

## Проблема хардкоду

```ts
// ❌ Погано: секрети та налаштування прямо в коді
@Injectable()
export class MailService {
  private readonly apiKey = 'sk_live_51H8xJ2...'; // потрапить у git-репозиторій!
  private readonly port = 3000;
}
```

## Рішення: ConfigModule + ConfigService

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

## Чому це важливо

- **Безпека:** секрети не потрапляють у git-репозиторій (`.env` додається в `.gitignore`).
- **Гнучкість середовищ:** dev/staging/prod використовують різні значення без зміни коду — лише різні `.env`-файли або змінні оточення.
- **DIP на рівні конфігурації:** сервіси залежать від абстракції `ConfigService`, а не від того, звідки саме взялося значення (файл, змінна оточення, secret-менеджер).

<!-- Місце для доповнення: валідація .env через Joi/class-validator (ConfigModule.forRoot({ validationSchema })), типізований конфіг через namespace-и (registerAs) -->
