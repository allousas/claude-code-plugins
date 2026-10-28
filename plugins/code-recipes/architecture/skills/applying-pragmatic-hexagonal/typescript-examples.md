# TypeScript Examples

## DDD style
```
src/
├── main.ts
├── domain/                  # Core business logic (pure, no framework dependencies): entities, value objects, domain events, domain errors, ports
│   ├── team.domain.ts       # Team aggregate, Member, TeamName
│   ├── team.events.ts       # TeamCreated, MemberAdded
│   ├── team.errors.ts       # TeamNotFoundException, AddMemberException
│   └── outbound.ports.ts    # TeamRepository, DomainEventPublisher
├── application/             # Use cases - orchestration of infrastructure and domain to provide business services
│   ├── use-cases/           # Command use cases (write operations)
│   │   ├── create-team.use-case.ts
│   │   └── add-team-member.use-case.ts
│   └── queries/             # Query handlers (read operations)
│       └── can-team-accept-member.query-handler.ts
└── infrastructure/          # Infrastructure adapters, thin translation layer
    ├── inbound/             # HTTP controllers, message consumers, schedulers, exception filters
    │   ├── team.controller.ts
    │   └── team-events.kafka-consumer.ts
    ├── outbound/            # Repositories, HTTP clients, message producers, event publisher/subscribers
    │   ├── postgres.team-repository.ts
    │   └── kafka.team-event-producer.ts
    ├── config/              # NestJS modules, providers, database connection
    │   └── app.module.ts
    └── infrastructure.errors.ts
```
Tests mirror `src/` under `test/` (`test/application/use-cases/create-team.use-case.test.ts`), shared fakes in `test/fixtures/`.

## By-the-book style
```
src/
├── main.ts
├── domain/
│   ├── model/               # Core business logic (pure, no framework dependencies): entities, value objects, domain events
│   │   ├── team.domain.ts
│   │   ├── team.events.ts
│   │   └── team.errors.ts
│   ├── ports/               # Abstract classes for outbound dependencies (repositories, external services)
│   │   └── outbound.ports.ts
│   └── use-cases/           # Use cases - orchestration of infrastructure and domain to provide business services
│       ├── create-team.use-case.ts
│       └── add-team-member.use-case.ts
└── infrastructure/          # Infrastructure adapters, thin translation layer
    ├── inbound/             # HTTP controllers, message consumers, schedulers
    │   └── team.controller.ts
    ├── outbound/            # Repositories, HTTP clients, message producers
    │   └── postgres.team-repository.ts
    └── config/              # NestJS modules, providers
        └── app.module.ts
```

## TypeScript / NestJS specifics
- Files: kebab-case with a role suffix (`create-team.use-case.ts`, `team.controller.ts`, `postgres.team-repository.ts`); classes PascalCase (`CreateTeamUseCase`); methods camelCase; use cases expose `execute()`
- No `Impl` suffix — name implementations descriptively (`PostgresTeamRepository`, not `TeamRepositoryImpl`)
- Group related types in one file instead of one file per class (`team.errors.ts`, `team.events.ts`, `outbound.ports.ts`); request/response DTOs used by a single adapter live in that adapter's file below the main class, extracted to `<feature>.dtos.ts` only once shared
- Outbound ports are `abstract class`es so they survive compilation as DI tokens; bind them in `config/` (`{provide: TeamRepository, useClass: PostgresTeamRepository}`)
- Accepted decorator in the application layer: `@Injectable()`; the domain carries no decorators and no `@nestjs/*` imports
- Import across layers with the `@/` path alias (`@/domain/outbound.ports`)
- Single package — no pnpm/npm workspaces per layer
