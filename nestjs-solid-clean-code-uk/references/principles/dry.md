# DRY (Don't Repeat Yourself)

## Суть принципу

Кожна частина знання або логіки повинна мати єдине, однозначне представлення в системі.

## Як застосовувати

Якщо ідентичний код з'являється у двох різних місцях — його варто винести у спільний сервіс, утиліту чи декоратор. У NestJS це найкраще реалізується через Custom Decorators, Shared Modules або Interceptors.

### Приклад: дублювання логіки → Custom Decorator

```ts
// ❌ Погано: та сама логіка діставання user.id дублюється в кожному контролері
@Get('profile')
getProfile(@Req() req: Request) {
  const userId = req.user.id;
  return this.usersService.findById(userId);
}

@Get('orders')
getOrders(@Req() req: Request) {
  const userId = req.user.id; // те саме дублюється
  return this.ordersService.findByUser(userId);
}
```

```ts
// ✅ Добре: логіка дістається один раз, у Custom Decorator
export const CurrentUserId = createParamDecorator(
  (data: unknown, ctx: ExecutionContext): number => {
    const request = ctx.switchToHttp().getRequest();
    return request.user.id;
  },
);
```

```ts
@Get('profile')
getProfile(@CurrentUserId() userId: number) {
  return this.usersService.findById(userId);
}

@Get('orders')
getOrders(@CurrentUserId() userId: number) {
  return this.ordersService.findByUser(userId);
}
```

### Приклад: дублювання відповіді → Interceptor

Якщо кілька ендпоінтів однаково обгортають відповідь (наприклад, у `{ data, timestamp }`) — цю логіку варто винести в один `Interceptor` (див. `references/cross-cutting/interceptors-pipes-guards.md`), а не повторювати в кожному контролері.

## Ризик: передчасна абстракція

Не варто робити все перевикористовуваним "про всяк випадок". Якщо дві частини коду виглядають однаково зараз, але мають різні причини для змін у майбутньому — краще тримати їх окремо, ніж створювати надто складну спільну абстракцію.

```ts
// ❌ Передчасна абстракція: "об'єднали" два методи, які лише зараз виглядають однаково
function validateEntity(entity: 'user' | 'order', data: any) {
  if (entity === 'user') {
    // правила валідації user, що розвиватимуться незалежно
  } else {
    // правила валідації order, що розвиватимуться незалежно
  }
}
```

Якщо `user` і `order` валідуються "однаково" лише випадково, а бізнес-правила для них розвиватимуться в різні боки — окремі `UserValidator` і `OrderValidator` виявляться простішими в підтримці, ніж одна "розумна" спільна функція з розгалуженнями всередині. DRY стосується дублювання **знання** (бізнес-правила), а не візуальної схожості коду.

<!-- Місце для доповнення: DRY на рівні DTO (успадкування через PartialType/PickType), межа між DRY і over-abstraction в Shared Module -->
