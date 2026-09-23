# Guards, Pipes та Interceptors

Архітектура NestJS побудована на цих трьох інструментах, які дозволяють відокремити крос-каттінг логіку (авторизація, валідація, трансформація відповіді) від бізнес-логіки контролерів і сервісів — це пряме застосування SRP на рівні всього застосунку.

## 1. Guards (Охоронці)

Guards відповідають за автентифікацію та авторизацію. Вони вирішують, чи слід пропускати запит до обробника (controller handler), базуючись на певній умові (наприклад, роль користувача чи наявність токена).

- **Коли працюють:** найпершими в ланцюжку запиту, одразу після Middleware.
- **Ключова функція:** `canActivate()` повертає `boolean` або `Promise<boolean>`. Якщо `false` — запит відхиляється (кидається виняток `ForbiddenException`).

```ts
@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private readonly reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRole = this.reflector.get<string>('role', context.getHandler());
    const request = context.switchToHttp().getRequest();
    return request.user?.role === requiredRole;
  }
}
```

```ts
@UseGuards(RolesGuard)
@Get('admin')
adminOnly() {
  // виконається, лише якщо canActivate() повернув true
}
```

## 2. Pipes (Труби)

Pipes використовуються для двох цілей:

1. **Трансформація даних** — приведення вхідних даних до потрібного формату (наприклад, рядок `"1"` → число `1`).
2. **Валідація** — перевірка вхідних даних на відповідність очікуваній схемі (зазвичай через `class-validator`, див. `references/data-validation/dto-validation.md`).

- **Коли працюють:** перед викликом методу обробника.
- **Ключова функція:** `transform()`.

```ts
@Injectable()
export class ParseIntPipe implements PipeTransform {
  transform(value: string): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException('Validation failed: not a number');
    }
    return val;
  }
}
```

```ts
@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  // сюди id вже потрапляє як гарантоване число
}
```

## 3. Interceptors (Перехоплювачі)

Найпотужніший з трьох інструментів — має доступ до запиту як до, так і після виконання методу обробника. Може змінювати результат запиту, обробляти помилки або додавати логіку логування.

- **Коли працюють:** огортають виконання методу обробника (тобто мають код "до" і код "після").
- **Ключові можливості:**
  - Перехоплення та модифікація відповіді (наприклад, обгортання результату в стандартну структуру `{ data, meta }`).
  - Обробка винятків (можна перехопити помилку і повернути власну відповідь).
  - Вимірювання часу виконання запиту.

```ts
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const start = Date.now();
    const request = context.switchToHttp().getRequest();

    return next.handle().pipe(
      tap(() => {
        console.log(`${request.method} ${request.url} — ${Date.now() - start}ms`);
      }),
    );
  }
}
```

```ts
@Injectable()
export class TransformResponseInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map((data) => ({ data, timestamp: new Date().toISOString() })),
    );
  }
}
```

`next.handle()` — це виклик самого методу-обробника; усе, що написано до нього в `intercept()`, виконується "до", а логіка всередині `.pipe(...)` — "після" завершення обробника.

## Коротке порівняння

| Інструмент | Питання, на яке відповідає |
|---|---|
| **Guards** | "Чи маєш ти доступ?" |
| **Pipes** | "Чи правильні вхідні дані?" |
| **Interceptors** | "Що зробити до і після виконання, і чи змінити результат?" |

## Порядок виконання в request lifecycle

```
Запит → Middleware → Guards → Interceptors (до) → Pipes → Controller → Service
      → Interceptors (після) → (Exception Filter, якщо стався виняток) → Відповідь
```

Guards виконуються раніше за Pipes і Interceptors не випадково: немає сенсу валідувати чи трансформувати дані запиту, до якого користувач взагалі не має доступу. А `Exception Filter` (див. `references/cross-cutting/exception-filters.md`) стоїть окремо від цього лінійного потоку — він перехоплює виняток, кинутий на будь-якому з попередніх етапів.

<!-- Місце для доповнення: глобальні vs локальні Guards/Pipes/Interceptors, кастомні декоратори параметрів (createParamDecorator), приклад ValidationPipe у зв'язці з whitelist -->
