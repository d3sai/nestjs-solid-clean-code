# Composition over Inheritance

## The essence of the principle

Object composition ("has-a") is preferable to inheritance ("is-a").

## How to apply it

Instead of deep class hierarchies (for example, `BaseUser -> RegisteredUser -> Admin`), use composition of services. In NestJS, this is naturally implemented through Dependency Injection.

### Example: deep hierarchy → composition

```ts
// ❌ Bad: a deep inheritance hierarchy
class BaseUser {
  login() { /* ... */ }
}

class RegisteredUser extends BaseUser {
  checkout() { /* ... */ }
}

class AdminUser extends RegisteredUser {
  banUser() { /* ... */ }
  // AdminUser drags along all the behavior of RegisteredUser and BaseUser,
  // even the parts the admin doesn't need or that should behave differently
}
```

```ts
// ✅ Good: composition via DI — behavior is assembled from separate parts
@Injectable()
export class UserService {
  constructor(
    private readonly authService: AuthService,
    private readonly checkoutService: CheckoutService,
    private readonly moderationService: ModerationService,
  ) {}

  login(credentials: LoginDto) {
    return this.authService.login(credentials);
  }

  banUser(userId: number) {
    return this.moderationService.banUser(userId);
  }
}
```

Each capability (`AuthService`, `CheckoutService`, `ModerationService`) is a separate, independently testable service. `UserService` only assembles the combination it needs through the constructor, rather than inheriting everything at once.

## Advantage

Composition allows an object's behavior to be assembled dynamically: dependencies (services, strategies, validators) are injected into the constructor, which gives a flexible system that's easy to test by swapping dependencies (mocking) — see `references/testing/testing-with-solid.md`.

This is also directly connected to `references/solid/lsp.md`: when an inheritance hierarchy is deep, subclasses often violate the base class's expectations (LSP) precisely because they've inherited too much of someone else's behavior. Composition removes this problem — a service depends only on what it actually uses.

## Summary: how all these principles work together

- **SOLID** sets the architectural structure.
- **DRY, KISS, YAGNI** control the quality and scope of the code.
- **Composition** provides flexibility in how components are connected.

<!-- Open for expansion: when inheritance is still appropriate (e.g. shared behavior among Exception classes), Mixins as a middle-ground option -->
