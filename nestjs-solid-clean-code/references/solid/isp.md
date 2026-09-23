# ISP — Interface Segregation Principle

## The essence of the principle

Large interfaces are best split into small ones, specific to the actual needs of their consumers. A client shouldn't be forced to depend on methods or fields it doesn't use.

## Example in NestJS

Avoid "fat" interfaces with many optional fields or methods that are actually needed by only some of the implementations:

```ts
// ❌ Bad: a "fat" interface forces implementing things that aren't needed
export interface FileStorage {
  upload(file: Buffer, path: string): Promise<string>;
  download(path: string): Promise<Buffer>;
  generateSignedUrl?(path: string): Promise<string>; // not every provider supports this
  listVersions?(path: string): Promise<string[]>;     // also not always needed
}
```

Instead, split it into small, focused interfaces, and a class implements only the ones it actually needs:

```ts
// ✅ Good: separate, narrow contracts
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
  // implements all three, since S3 supports all these capabilities
}

@Injectable()
export class LocalStorageProvider implements FileUploader, FileDownloader {
  // not forced to implement generateSignedUrl(), which it doesn't need
}
```

Services that consume these interfaces depend only on what they truly need (for example, a file-upload component depends only on `FileUploader`, not on the whole `FileStorage`).

<!-- Open for expansion: an example with DTO interfaces, splitting large service interfaces in a real project -->
