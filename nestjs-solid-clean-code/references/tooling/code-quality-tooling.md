# Code Quality Tooling

## Core Idea

Architectural principles (SOLID, DRY, KISS) define how to write code correctly, but they don't stop someone from accidentally committing code with inconsistent formatting or an obvious mistake. Automated tools that run before code lands in a shared branch cover these risks.

## ESLint + Prettier

- **ESLint** — static code analysis: finds potential errors (unused variables, unnecessary `any`, violations of project rules).
- **Prettier** — automatic formatting: eliminates "tabs vs. spaces"-style arguments from code review, because the tool simply unifies the style.

```json
// .eslintrc already ships with the NestJS CLI template by default; a typical addition:
{
  "rules": {
    "@typescript-eslint/no-unused-vars": "error",
    "@typescript-eslint/explicit-function-return-type": "off"
  }
}
```

## Husky + lint-staged: Pre-Commit Hooks

Runs the linter and formatter automatically before every commit — only on changed files, so the check stays fast:

```json
// package.json
{
  "lint-staged": {
    "*.ts": ["eslint --fix", "prettier --write"]
  }
}
```

```bash
npx husky init
echo "npx lint-staged" > .husky/pre-commit
```

If a file fails the linter, the commit is blocked locally — before the problematic code ever reaches the repository or CI.

## Conventional Commits

A standardized commit message format (`feat:`, `fix:`, `refactor:`, `chore:`) enables automatic CHANGELOG generation and semantic-version determination in the CI/CD pipeline:

```
feat(orders): add idempotency key support for order creation
fix(auth): correctly reject expired refresh tokens
refactor(users): extract notification logic into separate service
```

## Why This Matters

- **Consistency without arguments.** Code style stops being a topic of discussion in code review — Prettier handles it automatically, letting the reviewer focus on logic instead of formatting.
- **Early error detection.** ESLint catches a portion of problems (unused imports, suspicious patterns) before runtime, long before they reach tests or production.
- **Team scaling.** The larger the team, the more expensive it becomes to go without these tools: without them, code style inevitably diverges between developers, and errors a linter would have caught instantly instead reach code review or even production.

<!-- Open for expansion: SonarQube/CodeClimate for repository-wide quality metrics, configuring ESLint for specific NestJS patterns (e.g., banning direct new Service()) -->
