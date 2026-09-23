# DRY (Don't Repeat Yourself)

## The essence of the principle

Every piece of knowledge or logic should have a single, unambiguous representation in the system.

## How to apply it

If identical code appears in two different places, it should be extracted into a shared service, utility, or decorator. In NestJS, this is best implemented through Custom Decorators, Shared Modules, or Interceptors.

### Example: duplicated logic → Custom Decorator

```ts
// ❌ Bad: the same logic for getting user.id is duplicated in every controller
@Get('profile')
getProfile(@Req() req: Request) {
  const userId = req.user.id;
  return this.usersService.findById(userId);
}

@Get('orders')
getOrders(@Req() req: Request) {
  const userId = req.user.id; // the same thing duplicated
  return this.ordersService.findByUser(userId);
}
```

```ts
// ✅ Good: the logic is extracted once, into a Custom Decorator
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

### Example: duplicated response shaping → Interceptor

If several endpoints wrap their response the same way (for example, in `{ data, timestamp }`), this logic should be extracted into a single `Interceptor` (see `references/cross-cutting/interceptors-pipes-guards.md`) rather than repeated in every controller.

## Risk: premature abstraction

Don't make everything reusable "just in case." If two pieces of code look the same right now but have different reasons to change in the future, it's better to keep them separate than to create an overly complex shared abstraction.

```ts
// ❌ Premature abstraction: two methods "merged" just because they currently look alike
function validateEntity(entity: 'user' | 'order', data: any) {
  if (entity === 'user') {
    // user validation rules, which will evolve independently
  } else {
    // order validation rules, which will evolve independently
  }
}
```

If `user` and `order` are validated "the same way" only by coincidence, and their business rules will evolve in different directions, then separate `UserValidator` and `OrderValidator` classes turn out to be easier to maintain than a single "clever" shared function with branching inside it. DRY is about duplication of **knowledge** (business rules), not visual similarity of code.

<!-- Open for expansion: DRY at the DTO level (inheritance via PartialType/PickType), the line between DRY and over-abstraction in a Shared Module -->
