# ISP — Interface Segregation Principle

## Суть принципу

Великі інтерфейси варто розбивати на дрібні, специфічні для конкретних потреб споживачів. Клієнт не повинен бути змушений залежати від методів чи полів, якими він не користується.

## Приклад у NestJS

Уникайте «товстих» інтерфейсів із багатьма опціональними полями чи методами, які реально потрібні лише частині реалізацій:

```ts
// ❌ Погано: "товстий" інтерфейс змушує реалізовувати зайве
export interface FileStorage {
  upload(file: Buffer, path: string): Promise<string>;
  download(path: string): Promise<Buffer>;
  generateSignedUrl?(path: string): Promise<string>; // не всі провайдери це вміють
  listVersions?(path: string): Promise<string[]>;     // теж не завжди потрібно
}
```

Замість цього — виділяємо дрібні, сфокусовані інтерфейси, і клас реалізує лише ті, що йому реально потрібні:

```ts
// ✅ Добре: окремі, вузькі контракти
export interface FileUploader {
  upload(file: Buffer, path: string): Promise<string>;
}

export interface FileDownloader {
  download(path: string): Promise<Buffer>;
}

export interface SignedUrlProvider {
  generateSignedUrl(path: string): Promise<string>;
}

@Injectable()
export class S3StorageProvider implements FileUploader, FileDownloader, SignedUrlProvider {
  // реалізує всі три, бо S3 підтримує всі можливості
}

@Injectable()
export class LocalStorageProvider implements FileUploader, FileDownloader {
  // не змушений імплементувати generateSignedUrl(), який йому не потрібен
}
```

Сервіси, що споживають ці інтерфейси, залежать лише від того, що їм справді потрібне (наприклад, компонент завантаження файлів залежить тільки від `FileUploader`, а не від усього `FileStorage`).

<!-- Місце для доповнення: приклад із DTO-інтерфейсами, розбиття великих сервісних інтерфейсів у реальному проєкті -->
