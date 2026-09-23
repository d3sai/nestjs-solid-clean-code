# KISS (Keep It Simple, Stupid)

## Суть принципу

Більшість систем працюють найкраще, якщо лишаються простими, а не ускладненими.

## Як застосовувати

Уникайте «розумного» коду (clever code) на користь «зрозумілого». Якщо метод містить 5 рівнів вкладеності `if/else` — його потрібно рефакторити. NestJS заохочує модульність, тому розбиття на дрібні сервіси — найкращий шлях до KISS.

### Приклад: глибока вкладеність → guard clauses

```ts
// ❌ Погано: 5 рівнів вкладеності, важко відстежити логіку
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
// ✅ Добре: guard clauses — раннє повернення прибирає вкладеність
function calculateDiscount(user: User, order: Order) {
  if (!user || !user.isActive) return 0;
  if (!order || order.amount <= 0) return 0;

  return user.isPremium ? order.amount * 0.2 : order.amount * 0.1;
}
```

Обидва варіанти роблять те саме, але другий читається зверху вниз без потреби тримати в голові кілька рівнів умов одночасно.

### Приклад: "розумний" код vs зрозумілий

```ts
// ❌ Clever code: компактно, але важко читати з першого погляду
const total = items.reduce((a, i) => a + (i.qty * i.price * (1 - (i.disc ?? 0))), 0);
```

```ts
// ✅ Зрозумілий код: трохи довше, зате очевидно, що відбувається
function calculateItemTotal(item: OrderItem): number {
  const discount = item.discount ?? 0;
  return item.quantity * item.price * (1 - discount);
}

const total = items.reduce((sum, item) => sum + calculateItemTotal(item), 0);
```

## Порада

Якщо іншому розробнику (або вам через пів року) потрібно більше 5 хвилин, щоб зрозуміти, що робить функція — вона занадто складна. Це простий, але надійний практичний тест на відповідність KISS.

<!-- Місце для доповнення: KISS проти надмірної абстракції (коли патерн з інших розділів — це вже не KISS), приклад рефакторингу великого сервісу на дрібні -->
