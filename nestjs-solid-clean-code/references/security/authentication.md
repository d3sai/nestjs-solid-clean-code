# Authentication

## The Essence of the Approach

Authentication answers the question "who are you?" — it verifies that the user is who they claim to be (as opposed to authorization, which answers "what are you allowed to do?" — see `references/security/authorization.md`). In NestJS, the de facto standard is `@nestjs/passport` with a JWT strategy.

## Password Hashing

A password is never stored in plain text. Hashing is done via `bcrypt` (or `argon2`) — with a "salt" built into the algorithm itself:

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

`passwordHash` — this is exactly why this field must never end up in `UserResponseDto` (see `references/data-validation/dto-validation.md`).

## Passport Strategies: Local + JWT

The typical setup uses two strategies: `LocalStrategy` verifies the login/password during sign-in, and `JwtStrategy` verifies the token on every subsequent request.

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
    // whatever is returned here ends up in request.user
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
    // only requests that pass JwtStrategy.validate() reach here
  }
}
```

## Access Token + Refresh Token

A short-lived access token (e.g., 15 minutes) reduces the risk if it's stolen, while a long-lived refresh token allows obtaining a new access token without logging in again:

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

The refresh token should be stored server-side (hashed, like a password) and it should be possible to revoke it (e.g., on logout or compromise) — otherwise issuing new access tokens becomes uncontrolled.

## Why This Matters

- Authentication is a `Guard` applied to every protected endpoint (`references/cross-cutting/interceptors-pipes-guards.md`), not logic scattered across controllers.
- `JwtStrategy.validate()` is executed automatically by Nest on every request with a valid token — which is exactly why `Guard` is the very first step in the request lifecycle (before Pipes and Interceptors).

<!-- Open for expansion: OAuth2/social login (Google, GitHub), httpOnly cookies vs. Authorization header for tokens, refresh token rotation -->
