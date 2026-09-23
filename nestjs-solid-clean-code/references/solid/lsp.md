# LSP — Liskov Substitution Principle

## The essence of the principle

Subclasses (or implementations) must be interchangeable with their base type without breaking the correctness of the program. If code that works with the base type breaks when a specific implementation is substituted in, the principle has been violated.

## Example in NestJS

In NestJS this is best achieved through `implements` (a contract/interface) rather than `extends` (implementation inheritance). `implements` guarantees that a class only fulfills the promised contract, without dragging along the parent class's behavior, which a subclass might unintentionally violate.

```ts
// ✅ Good: a contract via interface + implements
export interface StorageProvider {
  upload(file: Buffer, path: string): Promise<string>;
}

@Injectable()
export class S3StorageProvider implements StorageProvider {
  async upload(file: Buffer, path: string): Promise<string> {
    // any implementation that honestly fulfills the upload() contract
    return 'https://s3.example.com/' + path;
  }
}

@Injectable()
export class LocalStorageProvider implements StorageProvider {
  async upload(file: Buffer, path: string): Promise<string> {
    // the local implementation also honestly fulfills the contract
    return '/uploads/' + path;
  }
}
```

Any code that expects a `StorageProvider` will work correctly regardless of which specific implementation was supplied.

```ts
// ❌ Risk of violating LSP: inheriting implementation via extends
class BaseRepository {
  async find(id: string) { /* ... */ }
}

class ReadOnlyRepository extends BaseRepository {
  async find(id: string) {
    throw new Error('Not supported'); // the subclass violates the base class's expectations
  }
}
```

If a subclass has to "throw an error" or otherwise forbid something the base class/interface allowed, that's a signal the hierarchy needs to be redesigned.

<!-- Open for expansion: more examples with NestJS Guards/Strategies, cases where extends is still appropriate -->
