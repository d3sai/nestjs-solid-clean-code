# YAGNI (You Ain't Gonna Need It)

## The essence of the principle

Don't implement functionality until it's actually needed.

## How to apply it

Don't write "flexible" code "just in case." Many developers create interfaces or configuration parameters for features that will never be implemented — this only clutters the codebase and makes it harder to maintain.

### Example

```ts
// ❌ Bad: "flexibility just in case" that nobody asked for
export interface NotificationOptions {
  channel: 'email' | 'sms' | 'push' | 'webhook' | 'slack'; // sms/push/webhook/slack aren't implemented yet
  retryStrategy?: 'linear' | 'exponential' | 'custom';      // no strategy is needed yet
  priority?: 'low' | 'normal' | 'high' | 'critical';        // nobody uses priorities
}

@Injectable()
export class NotificationService {
  send(options: NotificationOptions) {
    // only 'email' is actually implemented, the rest is dead code
  }
}
```

```ts
// ✅ Good: the minimal solution needed for today's actual requirement
@Injectable()
export class NotificationService {
  sendEmail(to: string, subject: string, body: string) {
    // ...
  }
}
```

When a real need for SMS or push notifications appears, that's when it will become clear which abstraction is actually needed (and whether it's needed at all), instead of guessing in advance.

## Philosophy

Write the minimal solution that works today. The need for scalability or flexibility will arise later — that's when it makes sense to refactor, based on actual requirements rather than assumptions.

## Relationship to other sections

YAGNI is the same reasoning behind the question of "is a Repository Pattern needed here" (`references/data-validation/repository-pattern.md`) or "is CQRS needed here" (`references/architecture/cqrs.md`): a complex abstraction is justified only when there's a real, not a hypothetical, reason for it.

<!-- Open for expansion: YAGNI versus "forward-looking design" — where's the line for when a bit of forward thinking is fine -->
