# Авторизація (Authorization)

## Суть підходу

Авторизація відповідає на питання "що тобі можна робити?" — вона завжди йде після автентифікації (`references/security/authentication.md`), яка встановлює, хто саме робить запит. У NestJS авторизація найчастіше реалізується через `Guard` (`references/cross-cutting/interceptors-pipes-guards.md`), але спосіб перевірки прав може ускладнюватись залежно від потреб проєкту.

## RBAC (Role-Based Access Control)

Найпростіша і найпоширеніша модель — права прив'язані до ролі користувача (`admin`, `manager`, `user`).

```ts
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

```ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!requiredRoles) return true; // ендпоінт без обмежень за ролями

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.includes(user.role);
  }
}
```

```ts
@UseGuards(AuthGuard('jwt'), RolesGuard)
@Roles('admin')
@Delete(':id')
remove(@Param('id') id: number) {
  // лише admin пройде RolesGuard
}
```

RBAC добре працює, доки прав небагато і вони грубозернисті ("адмін може все, юзер — лише своє"). Коли з'являються нюанси на кшталт "менеджер може редагувати замовлення лише свого відділу" — рольової моделі стає замало.

## ABAC / Policy-based (наприклад, CASL)

Атрибутивний підхід перевіряє права на основі властивостей конкретного об'єкта та контексту, а не лише ролі користувача:

```ts
export function defineAbilityFor(user: User) {
  const { can, build } = new AbilityBuilder(createMongoAbility);

  if (user.role === 'admin') {
    can('manage', 'all');
  } else {
    can('read', 'Order');
    can('update', 'Order', { ownerId: user.id }); // лише свої замовлення
  }

  return build();
}
```

```ts
@Injectable()
export class PoliciesGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const { user } = context.switchToHttp().getRequest();
    const order = context.switchToHttp().getRequest().order;
    const ability = defineAbilityFor(user);
    return ability.can('update', subject('Order', order));
  }
}
```

Це дозволяє висловити правило "менеджер може редагувати замовлення, лише якщо він його власник" — те, що чистий RBAC виразити природно не може.

## Коли який підхід обирати

- **RBAC** — достатньо для більшості застосунків з невеликою кількістю ролей і грубими правами. Простіше в реалізації й тестуванні.
- **ABAC/policy-based** — потрібен, коли права залежать від даних конкретного запису (власник, статус, відділ) чи від комбінації кількох умов. Дорожче в підтримці, тому впроваджується тоді, коли RBAC реально починає бракувати (YAGNI, `references/principles/yagni.md`).

## Чому це важливо

Авторизація, розкидана по `if`-перевірках всередині сервісів, — порушення SRP (`references/solid/srp.md`) і ризик забути перевірку в новому ендпоінті. Винесення в `Guard`/policy-шар робить правила доступу видимими, централізованими й такими, що тестуються окремо від бізнес-логіки.

<!-- Місце для доповнення: перевірка прав на рівні поля (field-level authorization) у GraphQL/REST-відповіді, аудит спроб доступу -->
