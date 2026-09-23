# DIP — Dependency Inversion Principle

## Суть принципу

Високорівневі модулі не повинні залежати від низькорівневих модулів — обидва мають залежати від абстракцій. Деталі реалізації (конкретний SDK, бібліотека, зовнішній сервіс) не повинні диктувати архітектуру бізнес-логіки.

## Приклад у NestJS

Бізнес-логіка не повинна напряму залежати від конкретного SDK, наприклад Amazon S3. Натомість вона залежить від абстракції (інтерфейсу), а конкретна реалізація підключається через Dependency Injection:

```ts
// ✅ Абстракція, від якої залежить високорівнева логіка
export interface StorageProvider {
  upload(file: Buffer, path: string): Promise<string>;
}

export const STORAGE_PROVIDER = Symbol('STORAGE_PROVIDER');

// Низькорівнева деталь реалізації — конкретний SDK
@Injectable()
export class S3StorageProvider implements StorageProvider {
  async upload(file: Buffer, path: string): Promise<string> {
    // виклики AWS S3 SDK
    return 'https://s3.example.com/' + path;
  }
}

// Високорівнева бізнес-логіка залежить лише від абстракції
@Injectable()
export class AvatarService {
  constructor(
    @Inject(STORAGE_PROVIDER) private readonly storage: StorageProvider,
  ) {}

  async uploadAvatar(file: Buffer, userId: string) {
    return this.storage.upload(file, `avatars/${userId}.png`);
  }
}
```

Підключення конкретної реалізації відбувається в модулі через `provide`/`useClass`:

```ts
@Module({
  providers: [
    { provide: STORAGE_PROVIDER, useClass: S3StorageProvider },
  ],
})
export class StorageModule {}
```

Завдяки цьому перехід з Amazon S3 на, наприклад, Google Cloud Storage зводиться до заміни одного провайдера (`useClass: GoogleCloudStorageProvider`) — без жодної зміни в `AvatarService` чи іншій бізнес-логіці, що його використовує.

## Порада: вбудований DI у NestJS

Nest має вбудовану систему Dependency Injection, яка природно підтримує DIP: залежності інжектуються через конструктор, а не створюються всередині класу (`new SomeService()`). Це саме той механізм, що робить заміну реалізацій (S3 → GCS, Stripe → PayPal тощо) простою та безпечною. Детальніше про механіку DI в Nest — у `references/dependency-injection/di-basics.md`.

<!-- Місце для доповнення: приклади з ConfigService, токени через InjectionToken, тестування через мок-провайдери -->
