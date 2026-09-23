# Межі модулів та Domain-Driven Design (DDD) Lite

## Суть підходу

Для великих проєктів код варто розділяти на **Bounded Contexts** (обмежені контексти) — самодостатні модулі, кожен зі своєю зоною відповідальності.

Наприклад, модуль `Payments` не повинен напряму знати про внутрішню структуру модуля `Users` (його Entities, приватні сервіси, схему БД). Спілкування між такими модулями має йти через явний, стабільний інтерфейс — публічний сервіс модуля або шину подій (Event Bus), а не через прямий імпорт внутрішніх класів іншого модуля.

## Приклад

```ts
// ❌ Погано: PaymentsService напряму лізе у внутрішню Entity іншого модуля
@Injectable()
export class PaymentsService {
  constructor(
    @InjectRepository(UserEntity) private readonly userRepo: Repository<UserEntity>,
  ) {}

  async charge(userId: number) {
    const user = await this.userRepo.findOne({ where: { id: userId } });
    // PaymentsService тепер залежить від деталей структури Users-модуля
  }
}
```

```ts
// ✅ Добре: спілкування через публічний сервіс іншого модуля
@Injectable()
export class PaymentsService {
  constructor(private readonly usersService: UsersService) {}

  async charge(userId: number) {
    const user = await this.usersService.getById(userId);
    // PaymentsService знає лише публічний контракт UsersService
  }
}
```

Ще слабший (і гнучкіший) варіант зв'язку — через шину подій, коли модулі взагалі не звертаються один до одного напряму, а лише публікують і слухають події:

```ts
// Users-модуль публікує подію, нічого не знаючи про Payments
this.eventEmitter.emit('user.created', { userId: user.id });

// Payments-модуль підписується на подію, нічого не знаючи про Users
@OnEvent('user.created')
handleUserCreated(payload: { userId: number }) {
  // ініціалізація платіжного профілю тощо
}
```

## Чому це важливо

Такий поділ — це те саме ISP та DIP, але на рівні модулів, а не окремих класів: модуль споживає лише вузький публічний контракт іншого модуля (аналог ISP) і залежить від абстракції взаємодії (сервіс/подія), а не від внутрішньої реалізації (аналог DIP). Це також зменшує ризик циклічних залежностей (`forwardRef`) між модулями.

<!-- Місце для доповнення: структура папок per-feature-модуль, приклад circular dependency та як його уникнути, коли DDD Lite виправданий, а коли over-engineering -->
