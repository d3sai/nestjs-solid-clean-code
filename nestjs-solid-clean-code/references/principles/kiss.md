# KISS (Keep It Simple, Stupid)

## The essence of the principle

Most systems work best if they stay simple rather than becoming complicated.

## How to apply it

Avoid "clever" code in favor of "clear" code. If a method contains 5 levels of `if/else` nesting, it needs to be refactored. NestJS encourages modularity, so splitting things into small services is the best path to KISS.

### Example: deep nesting → guard clauses

```ts
// ❌ Bad: 5 levels of nesting, hard to follow the logic
function calculateDiscount(user: User, order: Order) {
  if (user) {
    if (user.isActive) {
      if (order) {
        if (order.amount > 0) {
          if (user.isPremium) {
            return order.amount * 0.2;
          } else {
            return order.amount * 0.1;
          }
        }
      }
    }
  }
  return 0;
}
```

```ts
// ✅ Good: guard clauses — early returns remove the nesting
function calculateDiscount(user: User, order: Order) {
  if (!user || !user.isActive) return 0;
  if (!order || order.amount <= 0) return 0;

  return user.isPremium ? order.amount * 0.2 : order.amount * 0.1;
}
```

Both versions do the same thing, but the second one reads top to bottom without needing to hold several levels of conditions in your head at once.

### Example: "clever" code vs clear code

```ts
// ❌ Clever code: compact, but hard to read at a glance
const total = items.reduce((a, i) => a + (i.qty * i.price * (1 - (i.disc ?? 0))), 0);
```

```ts
// ✅ Clear code: a bit longer, but obvious what's happening
function calculateItemTotal(item: OrderItem): number {
  const discount = item.discount ?? 0;
  return item.quantity * item.price * (1 - discount);
}

const total = items.reduce((sum, item) => sum + calculateItemTotal(item), 0);
```

## Tip

If another developer (or you, six months from now) needs more than 5 minutes to understand what a function does, it's too complex. This is a simple but reliable practical test for KISS compliance.

<!-- Open for expansion: KISS versus excessive abstraction (when a pattern from other sections is no longer KISS), an example of refactoring a large service into small ones -->
