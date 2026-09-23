# Composition over Inheritance

## Суть принципу

Композиція об'єктів («має-а» — has-a) є кращою за наслідування («є-а» — is-a).

## Як застосовувати

Замість глибоких ієрархій класів (наприклад, `BaseUser -> RegisteredUser -> Admin`), використовуйте композицію сервісів. У NestJS це природно реалізується через Dependency Injection.

### Приклад: глибока ієрархія → композиція

```ts
// ❌ Погано: глибока ієрархія успадкування
class BaseUser {
  login() { /* ... */ }
}

class RegisteredUser extends BaseUser {
  checkout() { /* ... */ }
}

class AdminUser extends RegisteredUser {
  banUser() { /* ... */ }
  // AdminUser тягне за собою всю поведінку RegisteredUser і BaseUser,
  // навіть якщо частина з неї адміну не потрібна або має поводитись інакше
}
```

```ts
// ✅ Добре: композиція через DI — поведінка збирається з окремих частин
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

Кожна можливість (`AuthService`, `CheckoutService`, `ModerationService`) — окремий, незалежно тестований сервіс. `UserService` лише збирає потрібну комбінацію через конструктор, а не успадковує все підряд.

## Перевага

Композиція дозволяє динамічно збирати поведінку об'єкта: залежності (сервіси, стратегії, валідатори) ін'єктуються в конструктор, що дає гнучку систему, яку легко тестувати через підміну залежностей (mocking) — див. `references/testing/testing-with-solid.md`.

Це також напряму пов'язано з `references/solid/lsp.md`: коли ієрархія успадкування глибока, підкласи часто порушують очікування базового класу (LSP) саме тому, що успадкували забагато чужої поведінки. Композиція цю проблему знімає — сервіс залежить лише від того, що реально використовує.

## Підсумок: як усі ці принципи працюють разом

- **SOLID** задає структуру архітектури.
- **DRY, KISS, YAGNI** контролюють якість і обсяг коду.
- **Composition** забезпечує гнучкість зв'язків між компонентами.

<!-- Місце для доповнення: коли успадкування все ж доречне (наприклад, спільна поведінка Exception-класів), Mixins як проміжний варіант -->
