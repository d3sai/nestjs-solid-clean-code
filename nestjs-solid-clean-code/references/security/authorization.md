# Authorization

## The Essence of the Approach

Authorization answers the question "what are you allowed to do?" — it always comes after authentication (`references/security/authentication.md`), which establishes who is actually making the request. In NestJS, authorization is most often implemented via `Guard` (`references/cross-cutting/interceptors-pipes-guards.md`), but the way permissions are checked can grow more complex depending on the project's needs.

## RBAC (Role-Based Access Control)

The simplest and most common model — permissions are tied to a user's role (`admin`, `manager`, `user`).

```ts
export const Roles = (...roles: string[]) => SetMetadata('roles', roles);
```

```ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!requiredRoles) return true; // endpoint with no role restrictions

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
  // only an admin will pass RolesGuard
}
```

RBAC works well as long as there aren't many permissions and they're coarse-grained ("an admin can do anything, a user can only do their own things"). Once nuances appear, like "a manager can edit orders only for their own department" — the role model alone is no longer enough.

## ABAC / Policy-based (e.g., CASL)

An attribute-based approach checks permissions based on the properties of a specific object and its context, not just the user's role:

```ts
export function defineAbilityFor(user: User) {
  const { can, build } = new AbilityBuilder(createMongoAbility);

  if (user.role === 'admin') {
    can('manage', 'all');
  } else {
    can('read', 'Order');
    can('update', 'Order', { ownerId: user.id }); // only their own orders
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

This makes it possible to express the rule "a manager can edit an order only if they own it" — something plain RBAC cannot express naturally.

## Which Approach to Choose, and When

- **RBAC** — sufficient for most applications with a small number of roles and coarse-grained permissions. Simpler to implement and test.
- **ABAC/policy-based** — needed when permissions depend on the data of a specific record (owner, status, department) or on a combination of several conditions. More expensive to maintain, so it's introduced only once RBAC genuinely starts falling short (YAGNI, `references/principles/yagni.md`).

## Why This Matters

Authorization scattered across `if`-checks inside services is a violation of SRP (`references/solid/srp.md`) and a risk of forgetting the check in a new endpoint. Extracting it into a `Guard`/policy layer makes access rules visible, centralized, and testable separately from business logic.

<!-- Open for expansion: field-level authorization in GraphQL/REST responses, auditing access attempts -->
