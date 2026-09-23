# LSP — Liskov Substitution Principle

## Суть принципу

Підкласи (або реалізації) повинні бути взаємозамінними зі своїм базовим типом без порушення коректності програми. Якщо код, що працює з базовим типом, ламається при підстановці конкретної реалізації — принцип порушено.

## Приклад у NestJS

У NestJS це найкраще реалізується через `implements` (контракт-інтерфейс), а не через `extends` (успадкування реалізації). `implements` гарантує, що клас лише виконує обіцяний контракт, не тягнучи за собою поведінку батьківського класу, яку підклас може ненавмисно порушити.

```ts
// ✅ Добре: контракт через interface + implements
export interface StorageProvider {
  upload(file: Buffer, path: string): Promise<string>;
}

@Injectable()
export class S3StorageProvider implements StorageProvider {
  async upload(file: Buffer, path: string): Promise<string> {
    // будь-яка реалізація, що чесно виконує контракт upload()
    return 'https://s3.example.com/' + path;
  }
}

@Injectable()
export class LocalStorageProvider implements StorageProvider {
  async upload(file: Buffer, path: string): Promise<string> {
    // локальна реалізація теж чесно виконує контракт
    return '/uploads/' + path;
  }
}
```

Будь-який код, що очікує `StorageProvider`, працюватиме коректно незалежно від того, яку саме реалізацію підставили.

```ts
// ❌ Ризик порушення LSP: успадкування реалізації через extends
class BaseRepository {
  async find(id: string) { /* ... */ }
}

class ReadOnlyRepository extends BaseRepository {
  async find(id: string) {
    throw new Error('Not supported'); // підклас порушує очікування базового класу
  }
}
```

Якщо підклас має "викидати помилку" або по-іншому забороняти те, що дозволяв базовий клас/інтерфейс — це сигнал, що ієрархію варто перепроєктувати.

<!-- Місце для доповнення: більше прикладів на NestJS Guards/Strategies, коли extends все ж доречний -->
