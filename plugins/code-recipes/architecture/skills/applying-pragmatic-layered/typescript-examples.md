# TypeScript Examples

## Standard style
```
src/
├── main.ts
├── business/               # Core business logic
│   ├── domain/             # Entities, value objects, domain events, domain errors: team.domain.ts, team.events.ts, team.errors.ts
│   └── services/           # Use cases — orchestration of infrastructure and domain: create-team.service.ts, add-team-member.service.ts
├── entrypoint/             # Inbound adapters — triggers that invoke business services: team.controller.ts, team-events.kafka-consumer.ts, exception.filters.ts
└── infrastructure/         # Outbound adapters — technical concerns
    ├── db/                 # Repositories: postgres.team-repository.ts, database.schema.ts
    ├── clients/            # HTTP clients (internal and external): http-client.people-finder.ts
    ├── observability/      # Logging, metrics via domain event subscribers: domain-event.subscribers.ts
    └── config/             # NestJS modules, framework wiring: app.module.ts, database.providers.ts
```
Tests mirror `src/` under `test/`, shared fakes in `test/fixtures/`.

## TypeScript / NestJS specifics
- Concrete `@Injectable()` classes are their own DI tokens — list them in the module `providers` and inject them by class, no abstract-class ports or `provide/useClass` bindings
- Accepted decorator in the business layer: `@Injectable()`; the domain carries no decorators and no `@nestjs/*` imports
- Mock concrete infrastructure classes in tests with `jest.Mocked<T>` / `jest.spyOn`
- Same naming conventions as the hexagonal layout: kebab-case files with a role suffix, PascalCase classes, no `Impl` suffix, related types grouped in one file
- Single package — no pnpm/npm workspaces per layer
