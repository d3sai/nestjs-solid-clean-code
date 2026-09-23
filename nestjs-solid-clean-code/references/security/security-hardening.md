# Security Hardening: Helmet, CORS, Rate Limiting, Secrets

## The Essence of the Approach

Authentication and authorization (`references/security/authentication.md`, `references/security/authorization.md`) protect against "who" and "what" is allowed to do. But a production application needs another layer of protection — at the HTTP transport level and in general data hygiene.

## Helmet: Basic HTTP Security Headers

`Helmet` sets a collection of HTTP headers that protect against common attacks (clickjacking, MIME-sniffing, XSS via headers):

```ts
// main.ts
import helmet from 'helmet';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.use(helmet());
  await app.listen(3000);
}
```

## CORS: An Explicit Allowlist, Not `*`

```ts
// ❌ Bad: allows any site on the internet to make requests
app.enableCors({ origin: '*' });
```

```ts
// ✅ Good: an explicit list of trusted origins
app.enableCors({
  origin: ['https://app.example.com', 'https://admin.example.com'],
  credentials: true,
});
```

`origin: '*'` combined with `credentials: true` (cookies, authorization headers) is a dangerous combination that most browsers will block anyway, but an explicit allowlist is a deliberate control, not something left to chance.

## Rate Limiting via `@nestjs/throttler`

Protects against brute-force attacks on login, DDoS on expensive endpoints, and API abuse:

```ts
@Module({
  imports: [
    ThrottlerModule.forRoot([{ ttl: 60000, limit: 10 }]), // 10 requests per minute
  ],
  providers: [{ provide: APP_GUARD, useClass: ThrottlerGuard }],
})
export class AppModule {}
```

For sensitive endpoints (login, password reset), it's worth setting a stricter limit than for the rest of the API:

```ts
@Throttle({ default: { limit: 3, ttl: 60000 } }) // only 3 login attempts per minute
@Post('login')
login(@Body() dto: LoginDto) { /* ... */ }
```

## Secrets and Sensitive Data

- **Never hardcode secrets** — they live in `.env` and `ConfigService` (`references/architecture/configuration.md`), not in the code.
- **Never return sensitive fields in a response** — `passwordHash`, tokens, internal payment system IDs must not end up in a response DTO (`references/data-validation/dto-validation.md`).
- **Never log sensitive data** — passwords, tokens, card numbers, and personal data must never end up in logs (more details in `references/operations/logging.md`). This also applies to error messages: a stack trace containing a SQL query with a plaintext password is a classic leak through logs.

### If a Secret Has Already Ended Up in Code or Git

"Removing the hardcoded value" and "eliminating the risk" are not the same thing. If a secret key (API key, database password, JWT secret) has ever been committed — simply deleting the line from the code does **not** make it safe again:

- The key remains in the git history forever (`git log -p`, anyone with access to the repository can retrieve it), even after a force-push or squash, until the history is deliberately rewritten.
- If the repository was ever public (or became public by mistake) — consider the key compromised forever: bots scan GitHub for such patterns (`sk_live_...`, `AKIA...`) continuously and automatically.

**What to do if a secret has already been "exposed":**

1. **Immediately revoke/reissue the key** in the provider's dashboard (Stripe, AWS, etc.) — this is the first action to take, even before cleaning up the code. A key that's been removed from the code but not revoked still works for anyone who already copied it.
2. Remove the hardcoded value from the code and move it into `.env`/a secret manager.
3. If needed — rewrite the git history (`git filter-repo`, etc.) and force-push, understanding that this does not undo step 1 (clones of the repository held by other people will not have their history updated).
4. In production environments — use a dedicated secret manager (AWS Secrets Manager, HashiCorp Vault, Doppler) instead of a bare `.env` file on the server, and validate required environment variables at application startup (for example, `ConfigModule.forRoot({ validationSchema: Joi.object({...}) })`), so a forgotten or empty secret is caught right at deploy time, not at runtime.

## Protection Against Injections

Using an ORM (Prisma/TypeORM) with parameterized queries inherently protects against SQL injection — as long as there are no raw queries built via string concatenation:

```ts
// ❌ Unsafe: string concatenation in a raw SQL query
this.dataSource.query(`SELECT * FROM users WHERE email = '${email}'`);
```

```ts
// ✅ Safe: a parameterized query
this.dataSource.query('SELECT * FROM users WHERE email = $1', [email]);
```

Validating input data via `class-validator` (`references/data-validation/dto-validation.md`) additionally filters out values that don't match the expected format, before they ever reach a database query.

## Why This Matters

None of these mechanisms "closes off" security completely on its own — they work as layers: DTO validation doesn't replace rate limiting, and Helmet doesn't replace authorization. A production application needs all these layers at once, each defending against its own class of attack.

<!-- Open for expansion: CSRF protection for cookie-based sessions, npm audit / Dependabot for dependency vulnerabilities, Content Security Policy -->
