# SRP — Single Responsibility Principle

## Суть принципу

Кожен клас або модуль має виконувати лише одну задачу і мати лише одну причину для зміни. Якщо клас відповідає одразу за кілька речей, зміна в одній із них ризикує зачепити іншу — код стає крихким і важким для тестування.

## Приклад у NestJS

Типова помилка: логіка відправки email-повідомлень живе всередині сервісу, який відповідає за роботу з даними.

```ts
// ❌ Погано: UserService відповідає і за дані, і за нотифікації
@Injectable()
export class UserService {
  constructor(private readonly userRepository: UserRepository) {}

  async register(dto: CreateUserDto) {
    const user = await this.userRepository.save(dto);
    // стороння відповідальність всередині сервісу даних
    await this.sendWelcomeEmail(user.email);
    return user;
  }

  private async sendWelcomeEmail(email: string) {
    // SMTP-логіка, шаблони листів...
  }
}
```

Правильний підхід — винести розсилку в окремий сервіс сповіщень (наприклад, `NotificationService`), а `UserService` лишити відповідальним лише за роботу з даними користувача:

```ts
// ✅ Добре: кожен сервіс відповідає за одну зону
@Injectable()
export class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly notificationService: NotificationService,
  ) {}

  async register(dto: CreateUserDto) {
    const user = await this.userRepository.save(dto);
    await this.notificationService.sendWelcomeEmail(user.email);
    return user;
  }
}

@Injectable()
export class NotificationService {
  async sendWelcomeEmail(email: string) {
    // вся логіка розсилки живе тут
  }
}
```

Оркестрацію виклику (що і в якому порядку відбувається) можна лишити в сервісі чи контролері — головне, щоб сама реалізація сповіщень не змішувалась із логікою даних.

<!-- Місце для доповнення: додаткові приклади, edge cases, коли SRP порушують на рівні контролерів/модулів -->
