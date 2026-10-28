# Examples

## Fixtures

### Test builder
```typescript
// test/fixtures/test-builders.ts
export function buildTeam({
    id = randomUUID(),
    name = 'Teletubbies',
    version = 0,
}: Partial<{id: string; name: string; version: number}> = {}): Team {
    return new Team(id, name, version);
}

export function buildAnnualLeave({
    id = randomUUID(),
    employeeId = randomUUID(),
    year = 2024,
    leaves = [],
    version = 0,
}: Partial<{id: string; employeeId: string; year: number; leaves: readonly Leave[]; version: number}> = {}): AnnualLeave {
    return new AnnualLeave(id, employeeId, year, leaves, version);
}
```

### Postgres test container (one per test run)
```typescript
// test/fixtures/postgres.container.ts
// globalSetup and globalTeardown are separate modules: the handle travels on globalThis.
type ContainerHolder = {__postgresContainer?: StartedPostgreSqlContainer};
const holder = (): ContainerHolder => globalThis as unknown as ContainerHolder;

export async function startPostgres(): Promise<string> {
    const container = await new PostgreSqlContainer('postgres:16-alpine')
        .withDatabase('mydomain')
        .withUsername('mydomain')
        .withPassword('mydomain')
        .start();
    holder().__postgresContainer = container;
    return container.getConnectionUri();
}

export async function stopPostgres(): Promise<void> {
    await holder().__postgresContainer?.stop();
    holder().__postgresContainer = undefined;
}
```
```typescript
// test/fixtures/global-setup.ts — relative imports only (moduleNameMapper is not applied here)
import {startPostgres} from './postgres.container';

module.exports = async function globalSetup(): Promise<void> {
    if (process.env.DATABASE_URL) return; // run against an already-running Postgres (CI service, local compose)
    process.env.DATABASE_URL = await startPostgres(); // propagated to test workers
};
```
```typescript
// test/fixtures/global-teardown.ts
import {stopPostgres} from './postgres.container';

module.exports = async function globalTeardown(): Promise<void> {
    await stopPostgres();
};
```
```typescript
// test/fixtures/postgres.ts — migrations, connection and cleanup shared by integration and component tests
const run = promisify(execFile);
const MIGRATIONS_DIR = join(__dirname, '../../db/migrations');

// apply migrations with the real dbmate binary, same path as `pnpm db:migrate`
export async function migrate(): Promise<void> {
    await run('dbmate', ['--migrations-dir', MIGRATIONS_DIR, '--no-dump-schema', 'up'], {
        env: {...process.env, DATABASE_URL: databaseUrl()},
    });
}

export function createTestDatabase(): Kysely<Database> {
    return new Kysely<Database>({
        dialect: new PostgresDialect({pool: new Pool({connectionString: process.env.DATABASE_URL})}),
    });
}

export async function truncate(db: Kysely<Database>, ...tables: string[]): Promise<void> {
    await sql`TRUNCATE ${sql.join(tables.map((table) => sql.table(table)))}`.execute(db);
}

// dbmate refuses to connect to the local container unless SSL is explicitly disabled.
function databaseUrl(): string {
    const url = new URL(process.env.DATABASE_URL ?? DEFAULT_DATABASE_URL);
    if (!url.searchParams.has('sslmode')) url.searchParams.set('sslmode', 'disable');
    return url.toString();
}
```
```javascript
// jest.integration.config.js (excerpt)
module.exports = {
    testRegex: 'test/.*\\.(integration|component)\\.test\\.ts$',
    globalSetup: '<rootDir>/test/fixtures/global-setup.ts',
    globalTeardown: '<rootDir>/test/fixtures/global-teardown.ts',
    testTimeout: 60000, // pulling the image on a cold cache takes far longer than a query
    clearMocks: true,
};
```

### Fake of a port
```typescript
// test/fixtures/fake-transaction-provider.ts
export class FakeTransactionProvider implements TransactionProvider {
    run<T>(work: (trx: Transaction) => Promise<T>): Promise<T> {
        return work({} as Transaction);
    }
}
```

## Domain Layer Tests

### Unit test with nested suites
```typescript
// test/domain/team.domain.test.ts
describe('Team', () => {

    describe('create', () => {

        it('should create a team', () => {
            const teamId = randomUUID();
            const teamName = 'Teletubbies';

            const result = Team.create(teamId, teamName);

            expect(result).toEqual(new Team(teamId, teamName, 0));
        });

        it('should fail creating a team when name is too long', () => {
            const teamName = 'a'.repeat(300);

            const createTeam = () => Team.create(randomUUID(), teamName);

            expect(createTeam).toThrow(TeamValidationException);
            expect(createTeam).toThrow(expect.objectContaining({errorType: 'TOO_LONG_NAME'}));
        });
    });
});
```

## Application Service Tests

### Unit test with mocked dependencies
```typescript
// test/application/use-cases/create-team.use-case.test.ts
describe('CreateTeamUseCase', () => {

    const teamRepository = {find: jest.fn(), save: jest.fn()};

    const idGenerator = {generate: jest.fn()};

    const domainEventPublisher = {publish: jest.fn()};

    const createTeam = new CreateTeamUseCase(teamRepository, domainEventPublisher, new FakeTransactionProvider(), idGenerator);

    it('should create a team successfully', async () => {
        const team = buildTeam();
        idGenerator.generate.mockReturnValue(team.id);
        teamRepository.save.mockResolvedValue(right(team));

        const result = await createTeam.execute({name: team.name});

        expect(result).toEqual(right({id: team.id}));
        expect(domainEventPublisher.publish).toHaveBeenCalledWith(new TeamCreated(team), expect.anything());
    });

    it('should fail when repository rejects duplicate name', async () => {
        idGenerator.generate.mockReturnValue(randomUUID());
        teamRepository.save.mockResolvedValue(left(TEAM_ALREADY_EXISTS));

        const result = await createTeam.execute({name: 'Existing Team'});

        expect(result).toEqual(left(TEAM_ALREADY_EXISTS));
    });
});
```

## Infrastructure Tests

### Repository integration test
```typescript
// test/infrastructure/outbound/postgres.annual-leave-repository.integration.test.ts
describe('PostgresAnnualLeaveRepository', () => {

    let db: Kysely<Database>;
    let repository: PostgresAnnualLeaveRepository;
    let transactionProvider: KyselyTransactionProvider;

    const save = (annualLeave: AnnualLeave) => transactionProvider.run((trx) => repository.save(annualLeave, trx));

    beforeAll(async () => {
        await migrate();
        db = createTestDatabase();
        repository = new PostgresAnnualLeaveRepository(db);
        transactionProvider = new KyselyTransactionProvider(db);
    });

    afterAll(async () => {
        await db.destroy();
    });

    beforeEach(async () => {
        await truncate(db, 'annual_leaves');
    });

    it('should save and find an annual leave', async () => {
        const annualLeave = buildAnnualLeave();

        await save(annualLeave);
        const result = await repository.findBy(annualLeave.employeeId, annualLeave.year);

        expect(result).toEqual({...annualLeave, version: 1});
    });

    it('should not find an annual leave when it does not exist', async () => {
        expect(await repository.findBy(randomUUID(), 2042)).toBeNull();
    });

    it('should throw OptimisticLockingException when version does not match', async () => {
        const annualLeave = buildAnnualLeave();
        await save(annualLeave);
        const staleVersion = buildAnnualLeave({...annualLeave, version: 3});

        await expect(save(staleVersion)).rejects.toThrow(OptimisticLockingException);
    });
});
```

### HTTP client integration test with msw
```typescript
// test/infrastructure/outbound/public-holidays.client.integration.test.ts
import {http, HttpResponse} from 'msw';
import {setupServer} from 'msw/node';

describe('PublicHolidaysClient', () => {

    const baseUrl = 'http://public-holidays.test';
    const server = setupServer();
    const client = new PublicHolidaysClient({url: baseUrl, timeoutMillis: 1000});

    beforeAll(() => server.listen({onUnhandledRequest: 'error'}));
    afterEach(() => server.resetHandlers());
    afterAll(() => server.close());

    it('should fetch public holidays', async () => {
        server.use(http.get(`${baseUrl}/api/v3/PublicHolidays/2024/DE`, () =>
            HttpResponse.json([{date: '2024-12-25', name: 'Christmas Day'}])));

        const result = await client.fetch(2024, 'DE');

        expect(result).toEqual(['12-25']);
    });
});
```

### Controller integration test
```typescript
// test/infrastructure/inbound/team.controller.test.ts
describe('TeamController', () => {

    const createTeamUseCase = {execute: jest.fn()};
    let app: INestApplication;

    beforeEach(async () => {
        const module = await Test.createTestingModule({
            controllers: [TeamController],
            providers: [
                {provide: CreateTeamUseCase, useValue: createTeamUseCase},
                {provide: APP_FILTER, useClass: GlobalExceptionFilter}, // same global filter as production
            ],
        }).compile();

        app = module.createNestApplication();
        await app.init();
    });

    afterEach(async () => {
        await app.close();
    });

    it('should create a team', async () => {
        createTeamUseCase.execute.mockResolvedValue(right({id: randomUUID()}));

        await request(app.getHttpServer()).post('/teams').send({name: 'Teletubbies'}).expect(201);
    });

    it('should return 400 when team name is invalid', async () => {
        createTeamUseCase.execute.mockRejectedValue(new TeamValidationException('TOO_LONG_NAME'));

        await request(app.getHttpServer()).post('/teams').send({name: 'a'.repeat(300)}).expect(400);
    });
});
```

## Component Tests

### Base setup
```typescript
// test/component/component-test.setup.ts
export const stubServer = setupServer(); // msw, stands in for the external HTTP services

export function setupComponentTest(): {app: () => INestApplication} {
    let app: INestApplication;

    beforeAll(async () => {
        // 'bypass' lets supertest's requests to the app itself through; external calls are stubbed per test
        stubServer.listen({onUnhandledRequest: 'bypass'});
        process.env.CLIENTS_INTERNAL_EMPLOYEES_BASE_URL = EMPLOYEES_BASE_URL;
        await migrate(); // DATABASE_URL comes from the shared container started in globalSetup

        const module = await Test.createTestingModule({imports: [AppModule]}).compile();
        app = module.createNestApplication();
        await app.init();
    });

    afterEach(() => stubServer.resetHandlers());

    afterAll(async () => {
        await app.close();
        stubServer.close();
    });

    return {app: () => app};
}
```

### Happy path E2E
```typescript
// test/component/request-leave.component.test.ts
describe('RequestLeaveComponentTest', () => {

    const {app} = setupComponentTest();
    const employeeId = randomUUID();

    it('should request a leave successfully', async () => {
        stubEmployeeExists(stubServer, {employeeId, countryCode: 'DE'});
        stubPublicHolidays(stubServer, {year: 2024, countryCode: 'DE'});

        await request(app().getHttpServer())
            .post('/annual-leaves')
            .send({employeeId, days: ['12-20'], year: 2024, type: 'VACATION'})
            .expect(201);
    });
});
```

## TypeScript / NestJS specifics
- Libraries: Jest, `@testcontainers/postgresql`, `supertest`, `msw` (or a WireMock container) for HTTP stubs, a real Kafka container for KafkaJS
- Files mirror the source path with a `.test.ts` suffix (`test/application/use-cases/create-team.use-case.test.ts`); integration and component tests use `.integration.test.ts` / `.component.test.ts` and run in a separate Jest config
- Name suites after the class (`describe('CreateTeamUseCase')`) and tests as sentences via `it('should ...')`; nest `describe` blocks to group
- Prefer a typed fake class implementing the port; cast minimal fakes with `as` only at the boundary (`work({} as Transaction)`)
- Reset mocks between tests (`clearMocks: true` in jest config, or `jest.clearAllMocks()` in `afterEach`) — `jest.fn()` objects declared at suite level are shared across `it` blocks
- Assert the concrete error class, not its message: `await expect(promise).rejects.toThrow(TeamNotFoundException)`
- Never `jest.mock` libraries you don't own (`pg`, `kysely`, `fetch`, `kafkajs`); use the real engine, a stub server, or Jest fake timers / an injected clock for time
- Start the database with `@testcontainers/postgresql` from Jest's `globalSetup` and stop it in `globalTeardown`; publish the connection string as `DATABASE_URL` (Jest propagates `process.env` from `globalSetup` to workers). Keep the container handle on `globalThis`
- Keep `globalSetup` and its imports on relative paths — `moduleNameMapper` is not reliably applied to setup scripts, so an `@/`-aliased import there fails to resolve
- Build the module under test with `Test.createTestingModule({...}).compile()`, then `module.createNestApplication()` + `await app.init()`; always `await app.close()`
- Replace real providers with test doubles via `useValue` (or `.overrideProvider(Token).useValue(...)`)
- To test global behavior (validation pipe, exception filter), register the same `APP_*` providers in the testing module so the request path matches production
- Prove filter mapping by throwing the domain error from a mocked use case and asserting the mapped status over `supertest` — never call the filter directly
- Component tests import the real `AppModule`; share their setup through a `setupComponentTest()` function that registers the lifecycle hooks (Jest has no base test classes)
