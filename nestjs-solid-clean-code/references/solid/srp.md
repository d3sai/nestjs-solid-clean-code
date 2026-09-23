# SRP — Single Responsibility Principle

## The essence of the principle

Each class or module should perform only one job and have only one reason to change. If a class is responsible for several things at once, a change in one of them risks affecting another — the code becomes fragile and hard to test.

## Example in NestJS

A typical mistake: the logic for sending email notifications lives inside a service that's responsible for working with data.

```ts
// ❌ Bad: UserService is responsible for both data and notifications
@Injectable()
export class UserService {
  constructor(private readonly userRepository: UserRepository) {}

  async register(dto: CreateUserDto) {
    const user = await this.userRepository.save(dto);
    // an unrelated responsibility inside the data service
    await this.sendWelcomeEmail(user.email);
    return user;
  }

  private async sendWelcomeEmail(email: string) {
    // SMTP logic, email templates...
  }
}
```

The right approach is to extract the mailing logic into a separate notification service (e.g. `NotificationService`), leaving `UserService` responsible only for working with user data:

```ts
// ✅ Good: each service is responsible for one area
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
    // all the mailing logic lives here
  }
}
```

Orchestrating the call (what happens and in what order) can stay in the service or controller — what matters is that the notification implementation itself isn't mixed with the data logic.

<!-- Open for expansion: additional examples, edge cases, where SRP gets violated at the controller/module level -->
