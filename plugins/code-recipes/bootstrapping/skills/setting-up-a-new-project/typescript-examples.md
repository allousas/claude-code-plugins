# TypeScript Examples

## Default Tech Stack

| Category | Default |
|----------|---------|
| Runtime / Language | Node.js LTS (pinned in `.nvmrc`), TypeScript 5 (`strict`) |
| Framework | NestJS 11 (`@nestjs/platform-express`) |
| Database | PostgreSQL (`postgres:16-alpine`) |
| DB Access | Kysely + `pg` |
| Migrations | dbmate (Go binary, `brew install dbmate` — not a pnpm dependency) |
| Package Manager | pnpm |
| Build | `tsc --noEmit` (typecheck) + tsup (production bundle) |
| Messaging | In-process `InMemoryDomainEventPublisher`; KafkaJS only once events must leave the service |
| Test Framework / Mocking / Assertions | Jest + ts-jest |
| Containers | `@testcontainers/postgresql` |
| API Testing | supertest + `@nestjs/testing` |
| Linting | eslint + `@typescript-eslint` |

## Default Architecture (hexagonal, DDD style)
```
src/
├── main.ts
├── domain/
├── application/use-cases/
└── infrastructure/inbound/ | outbound/ | config/
db/migrations/
test/                      # mirrors src/, plus test/fixtures/
```

## Required Input
- **Service name** (e.g. `team-service`) — used as the `package.json` name and Kafka `clientId`

## package.json scripts
```json
{
  "scripts": {
    "typecheck": "tsc --noEmit",
    "build": "tsc --noEmit && tsup",
    "start": "node dist/main.js",
    "start:dev": "nest start --watch",
    "start:debug": "nest start --debug --watch",
    "lint": "eslint \"src/**/*.ts\"",
    "test": "jest",
    "test:cov": "jest --coverage",
    "test:integration": "jest --config jest.integration.config.js",
    "db:up": "docker compose up -d --wait",
    "db:down": "docker compose down -v",
    "db:migrate": "dbmate --migrations-dir ./db/migrations --no-dump-schema up",
    "db:rollback": "dbmate --migrations-dir ./db/migrations --no-dump-schema down",
    "db:status": "dbmate --migrations-dir ./db/migrations status",
    "db:new": "dbmate --migrations-dir ./db/migrations new"
  }
}
```

## Config files
- `tsconfig.json` — `strict`, `experimentalDecorators`, `emitDecoratorMetadata`, `target: ES2022`, path alias `"@/*": ["src/*"]`; `tsconfig.test.json` for tests
- `tsup.config.ts` — entry `src/main.ts`, `cjs`, sourcemaps, `@` alias resolved to `src`
- `jest.config.js` — unit tests `test/**/*.test.ts`, ignores `*.integration.test.ts`, `@/` mapped via `moduleNameMapper`
- `jest.integration.config.js` — `*.integration.test.ts`, `globalSetup`/`globalTeardown` start one Postgres container per run and apply migrations with dbmate; honours an existing `DATABASE_URL`
- `docker-compose.yml` — Postgres with a healthcheck (so `docker compose up --wait` blocks until ready)
- `.env.example` — `DATABASE_URL=postgres://<svc>:<svc>@localhost:5433/<svc>?sslmode=disable` (read by dbmate only; the app falls back to a default URL)
- `.eslintrc.js`, `nest-cli.json`, `.nvmrc`

## TypeScript / NestJS specifics
- Single package, no workspaces
- `main.ts` calls `app.enableShutdownHooks()` so the Kysely pool is closed on shutdown
- Integration tests need Docker and the dbmate binary on the PATH
