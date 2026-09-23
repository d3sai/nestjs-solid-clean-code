# Автентифікація (Authentication)

## Суть підходу

Автентифікація відповідає на питання "хто ти?" — перевіряє, що користувач той, за кого себе видає (на відміну від авторизації, яка відповідає "що тобі можна?" — див. `references/security/authorization.md`). У NestJS стандарт де-факто — `@nestjs/passport` з JWT-стратегією.

## Хешування паролів

Пароль ніколи не зберігається у відкритому вигляді. Хешування виконується через `bcrypt` (чи `argon2`) — з "сіллю", вбудованою в сам алгоритм:

```ts
@Injectable()
export class AuthService {
  async register(dto: RegisterDto) {
    const passwordHash = await bcrypt.hash(dto.password, 10);
    return this.usersService.create({ ...dto, passwordHash });
  }

  async validateUser(email: string, password: string) {
    const user = await this.usersService.findByEmail(email);
    if (!user || !(await bcrypt.compare(password, user.passwordHash))) {
      return null;
    }
    return user;
  }
}
```

`passwordHash` — саме тому це поле ніколи не потрапляє в `UserResponseDto` (див. `references/data-validation/dto-validation.md`).

## Passport-стратегії: Local + JWT

Типова схема — дві стратегії: `LocalStrategy` перевіряє логін/пароль під час входу, `JwtStrategy` перевіряє токен на кожному подальшому запиті.

```ts
@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private readonly authService: AuthService) {
    super({ usernameField: 'email' });
  }

  async validate(email: string, password: string) {
    const user = await this.authService.validateUser(email, password);
    if (!user) throw new UnauthorizedException();
    return user;
  }
}
```

```ts
@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private readonly configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      secretOrKey: configService.get<string>('JWT_SECRET'),
    });
  }

  async validate(payload: { sub: number; email: string }) {
    // те, що повертається тут, потрапляє в request.user
    return { id: payload.sub, email: payload.email };
  }
}
```

```ts
@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @UseGuards(AuthGuard('local'))
  @Post('login')
  login(@Req() req: Request) {
    return this.authService.issueTokens(req.user);
  }
}

@Controller('orders')
export class OrdersController {
  @UseGuards(AuthGuard('jwt'))
  @Get()
  findAll(@Req() req: Request) {
    // сюди дійде лише той, хто пройшов JwtStrategy.validate()
  }
}
```

## Access token + Refresh token

Короткоживучий access token (наприклад, 15 хвилин) знижує ризик, якщо його викрадуть, а довгоживучий refresh token дозволяє отримати новий access token без повторного логіну:

```ts
@Injectable()
export class AuthService {
  issueTokens(user: { id: number; email: string }) {
    const payload = { sub: user.id, email: user.email };
    return {
      accessToken: this.jwtService.sign(payload, { expiresIn: '15m' }),
      refreshToken: this.jwtService.sign(payload, { expiresIn: '7d' }),
    };
  }
}
```

Refresh token варто зберігати на стороні сервера (хешованим, як пароль) і мати можливість його відкликати (наприклад, при виході з системи чи компрометації) — інакше видача нового access token стає неконтрольованою.

## Чому це важливо

- Автентифікація — це `Guard`, застосований до кожного захищеного ендпоінта (`references/cross-cutting/interceptors-pipes-guards.md`), а не логіка, розкидана по контролерах.
- `JwtStrategy.validate()` виконується Nest автоматично на кожен запит із валідним токеном — саме тому `Guard` є найпершим у request lifecycle (перед Pipes та Interceptors).

<!-- Місце для доповнення: OAuth2/Social login (Google, GitHub), httpOnly cookies vs Authorization header для токенів, ротація refresh token -->
