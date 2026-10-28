# Examples

## NestJS provider factory for http
```typescript
// infrastructure/config/http-client.providers.ts
export interface PeopleServiceConfig {
    readonly url: string;
    readonly connectTimeoutMillis: number;
    readonly readTimeoutMillis: number;
}

export const PEOPLE_SERVICE_CONFIG = Symbol('PeopleServiceConfig');

export const peopleServiceConfigProvider: Provider = {
    provide: PEOPLE_SERVICE_CONFIG,
    useFactory: (): PeopleServiceConfig => ({
        url: requiredEnv('CLIENTS_PEOPLE_SERVICE_URL'),
        connectTimeoutMillis: Number(process.env.CLIENTS_PEOPLE_SERVICE_CONNECT_TIMEOUT_MILLIS ?? 1000),
        readTimeoutMillis: Number(process.env.CLIENTS_PEOPLE_SERVICE_READ_TIMEOUT_MILLIS ?? 2000),
    }),
};

export const peopleServiceApiProvider: Provider = {
    provide: PeopleServiceApi,
    inject: [PEOPLE_SERVICE_CONFIG],
    useFactory: (config: PeopleServiceConfig): PeopleServiceApi =>
        new PeopleServiceApi({
            baseUrl: config.url,
            dispatcher: new Agent({ // undici
                connect: {timeout: config.connectTimeoutMillis},
                headersTimeout: config.readTimeoutMillis,
                bodyTimeout: config.readTimeoutMillis,
            }),
            // opossum options: only 5xx responses count as failures that open the circuit
            circuitBreaker: {errorFilter: (error: unknown) => !isServerError(error)},
        }),
};

// HttpPeopleFinder (the adapter for the PeopleFinder port) is a plain @Injectable() with constructor injection
export const peopleFinderProvider: Provider = {provide: PeopleFinder, useClass: HttpPeopleFinder};

function isServerError(error: unknown): boolean {
    return error instanceof HttpCallNonSucceededException && error.httpStatus >= 500;
}

function requiredEnv(name: string): string {
    const value = process.env[name];
    if (!value) throw new Error(`Missing required environment variable ${name}`);
    return value;
}
```

## Constructor injection with defaults for testability
```typescript
// domain/outbound.ports.ts
export abstract class IdGenerator {
    abstract generate(): string;
}
```
```typescript
// application/use-cases/create-team.use-case.ts
@Injectable()
export class CreateTeamUseCase {
    constructor(
        private readonly teamRepository: TeamRepository,
        private readonly domainEventPublisher: DomainEventPublisher,
        private readonly idGenerator: IdGenerator, // production default bound in AppModule, override in tests
    ) {}
}
```
```typescript
// infrastructure/config/app.module.ts (excerpt) — the production default
{provide: IdGenerator, useValue: {generate: randomUUID}},
```

## Port as an abstract class (runtime DI token)
```typescript
// domain/outbound.ports.ts
export abstract class TeamRepository {
    abstract find(id: string): Promise<Team | null>;
    abstract save(team: Team, trx: Transaction): Promise<void>;
}
```
```typescript
// use case — plain constructor injection, no @Inject needed
constructor(private readonly teamRepository: TeamRepository) {}
```

## Module binding ports to adapters and global providers
```typescript
// infrastructure/config/app.module.ts
@Module({
    controllers: [HealthController, TeamController],
    providers: [
        CreateTeamUseCase,
        AddTeamMemberUseCase,
        kyselyProvider,
        DatabaseConnection,
        peopleServiceConfigProvider,
        peopleServiceApiProvider,
        peopleFinderProvider,
        {provide: TeamRepository, useClass: PostgresTeamRepository},
        {provide: TransactionProvider, useClass: KyselyTransactionProvider},
        {provide: DomainEventPublisher, useClass: InMemoryDomainEventDispatcher},
        {provide: IdGenerator, useValue: {generate: randomUUID}},
        {provide: APP_PIPE, useClass: ValidationPipe},
        {provide: APP_FILTER, useClass: GlobalExceptionFilter}, // see handling-errors-with-exceptions
    ],
})
export class AppModule {}
```

## Grouped provider definitions (database)
```typescript
// infrastructure/config/database.providers.ts
export const DEFAULT_DATABASE_URL = 'postgres://mydomain:mydomain@localhost:5432/mydomain';

// `new Pool` does not connect until the first query, so this never blocks startup on the database.
export const kyselyProvider: Provider = {
    provide: Kysely,
    useFactory: () => new Kysely<Database>({
        dialect: new PostgresDialect({
            pool: new Pool({connectionString: process.env.DATABASE_URL ?? DEFAULT_DATABASE_URL}),
        }),
    }),
};

// Closes the pool on shutdown; requires `app.enableShutdownHooks()` in `main.ts`.
@Injectable()
export class DatabaseConnection implements BeforeApplicationShutdown {
    constructor(private readonly db: Kysely<Database>) {}

    async beforeApplicationShutdown(): Promise<void> {
        await this.db.destroy();
    }
}
```

## TypeScript / NestJS specifics
- **TypeScript interfaces are erased at compile time — they cannot be DI tokens.** Injecting a bare interface type emits `Object` as the design type, and Nest fails at boot with *"Nest can't resolve dependencies"*. A port needs a value that exists at runtime:
  1. **Preferred — `abstract class` as the port.** Exists at runtime, doubles as type *and* token, needs no `@Inject`, and stays framework-free
  2. **Alternative — `interface` + `string`/`Symbol` token** (`export const TEAM_REPOSITORY = Symbol('TeamRepository')`, then `@Inject(TEAM_REPOSITORY)` at every call site). Keeps the domain as pure interfaces at the cost of explicit `@Inject`

  Pick one convention and apply it to every port
- Default constructor parameters (Kotlin's `generateId: () -> UUID = ...` style) don't work with Nest DI — it tries to resolve every constructor parameter. Model id generation / clock as a port (`IdGenerator`, `Clock`) bound to its production default in the module; tests pass a fake to the constructor
- Keep all wiring in modules and provider files under `infrastructure/config`
- Bind port → adapter with `useClass`; `useValue` for fakes/constants; `useFactory` (with `inject`) when a provider needs runtime construction (e.g. a Kysely pool from env, an HTTP client with timeouts and a circuit breaker)
- Register global cross-cutting providers via the `APP_*` tokens (`APP_PIPE`, `APP_FILTER`, `APP_INTERCEPTOR`) so they can themselves use DI. Don't register a global filter both via `APP_FILTER` and `app.useGlobalFilters()` — pick one
- Read environment variables only in configuration providers and expose them as typed config objects (`PeopleServiceConfig`); a single value can be read directly where it is used in the provider
- If a provider must be visible to other modules, add it to `exports`
- Simple adapters are just `@Injectable()` classes with constructor injection; reserve `useFactory` for components that need programmatic construction
- Never use the `Impl` suffix for adapters — name them by what distinguishes them (`PostgresTeamRepository`)
