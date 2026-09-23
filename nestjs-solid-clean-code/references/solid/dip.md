# DIP — Dependency Inversion Principle

## The essence of the principle

High-level modules shouldn't depend on low-level modules — both should depend on abstractions. Implementation details (a specific SDK, library, external service) shouldn't dictate the architecture of the business logic.

## Example in NestJS

Business logic shouldn't depend directly on a specific SDK, such as Amazon S3. Instead, it depends on an abstraction (an interface), and the concrete implementation is wired in via Dependency Injection:

```ts
// ✅ The abstraction that the high-level logic depends on
export interface StorageProvider {
  upload(file: Buffer, path: string): Promise<string>;
}

export const STORAGE_PROVIDER = Symbol('STORAGE_PROVIDER');

// Low-level implementation detail — a specific SDK
@Injectable()
export class S3StorageProvider implements StorageProvider {
  async upload(file: Buffer, path: string): Promise<string> {
    // AWS S3 SDK calls
    return 'https://s3.example.com/' + path;
  }
}

// High-level business logic depends only on the abstraction
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

The concrete implementation is wired in at the module level via `provide`/`useClass`:

```ts
@Module({
  providers: [
    { provide: STORAGE_PROVIDER, useClass: S3StorageProvider },
  ],
})
export class StorageModule {}
```

Thanks to this, migrating from Amazon S3 to, say, Google Cloud Storage comes down to swapping a single provider (`useClass: GoogleCloudStorageProvider`) — without any change to `AvatarService` or other business logic that uses it.

## Tip: built-in DI in NestJS

Nest has a built-in Dependency Injection system that naturally supports DIP: dependencies are injected through the constructor rather than created inside the class (`new SomeService()`). This is exactly the mechanism that makes swapping implementations (S3 → GCS, Stripe → PayPal, etc.) simple and safe. For more on how DI works in Nest, see `references/dependency-injection/di-basics.md`.

<!-- Open for expansion: examples with ConfigService, tokens via InjectionToken, testing with mock providers -->
