# Module Boundaries and Domain-Driven Design (DDD) Lite

## Core Idea

For large projects, code should be split into **Bounded Contexts** — self-contained modules, each with its own area of responsibility.

For example, the `Payments` module should not directly know about the internal structure of the `Users` module (its Entities, private services, database schema). Communication between such modules should go through an explicit, stable interface — a module's public service or an event bus — rather than through a direct import of another module's internal classes.

## Example

```ts
// ❌ Bad: PaymentsService reaches directly into another module's internal Entity
@Injectable()
export class PaymentsService {
  constructor(
    @InjectRepository(UserEntity) private readonly userRepo: Repository<UserEntity>,
  ) {}

  async charge(userId: number) {
    const user = await this.userRepo.findOne({ where: { id: userId } });
    // PaymentsService now depends on the internal structure details of the Users module
  }
}
```

```ts
// ✅ Good: communication through another module's public service
@Injectable()
export class PaymentsService {
  constructor(private readonly usersService: UsersService) {}

  async charge(userId: number) {
    const user = await this.usersService.getById(userId);
    // PaymentsService only knows UsersService's public contract
  }
}
```

An even looser (and more flexible) form of coupling is through an event bus, where modules never call each other directly and instead only publish and listen for events:

```ts
// The Users module publishes an event, knowing nothing about Payments
this.eventEmitter.emit('user.created', { userId: user.id });

// The Payments module subscribes to the event, knowing nothing about Users
@OnEvent('user.created')
handleUserCreated(payload: { userId: number }) {
  // initializing a payment profile, etc.
}
```

## Why This Matters

This separation is the same as ISP and DIP, but at the module level rather than the individual class level: a module consumes only a narrow public contract of another module (an analogue of ISP) and depends on an abstraction of the interaction (a service/event) rather than on the internal implementation (an analogue of DIP). It also reduces the risk of circular dependencies (`forwardRef`) between modules.

<!-- Open for expansion: per-feature-module folder structure, an example of a circular dependency and how to avoid it, when DDD Lite is justified versus over-engineering -->
