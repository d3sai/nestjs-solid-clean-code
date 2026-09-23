# NestJS SOLID & Clean Code

A Claude Code skill for writing production-grade NestJS and TypeScript code: the five SOLID principles, plus DRY, KISS, YAGNI, layered architecture, dependency injection, DTOs and validation, security, logging, caching, and testing.

## What's inside

- `SKILL.md`: the entry point, with a quick-reference table pointing to the right file for the task at hand
- `references/solid/`: SRP, OCP, LSP, ISP, DIP, each with before/after NestJS examples
- `references/principles/`: DRY, KISS, YAGNI, composition over inheritance
- `references/architecture/`: layered architecture, module boundaries, CQRS, event-driven design, transactions, configuration
- `references/dependency-injection/`: DI basics, common anti-patterns, request scoping
- `references/data-validation/`: DTOs vs entities, the repository pattern (and when it's overkill)
- `references/cross-cutting/`: exception filters, guards, pipes, interceptors
- `references/security/`: authentication, authorization, hardening
- `references/operations/`: logging, caching, background jobs, migrations, idempotency, health checks
- `references/testing/`: unit tests with SOLID, the testing pyramid
- `references/tooling/`: linting, formatting, pre-commit hooks

## Install

```bash
npx skills add <owner>/<repo> --skill nestjs-solid-clean-code
```

## Also in Ukrainian

The original version lives in `nestjs-solid-clean-code-uk/`.
