# TypeScript Examples

All based on NestJS, if the project uses any other framework, use the equivalent providers/decorators.

## Simple use case with NestJS
```typescript
// src/application/use-cases/create-team.use-case.ts
@Injectable()
export class CreateTeamUseCase {

    constructor(
        private readonly teamRepository: TeamRepository,
        private readonly transactionProvider: TransactionProvider,
        private readonly domainEventPublisher: DomainEventPublisher,
        private readonly idGenerator: IdGenerator,
    ) {}

    execute(teamName: string): Promise<string> {
        return this.transactionProvider.run(async (trx) => {
            const newTeam = Team.create(this.idGenerator.generate(), teamName);

            await this.teamRepository.save(newTeam, trx);
            await this.domainEventPublisher.publish(new TeamCreated(newTeam), trx);

            return newTeam.id;
        });
    }
}
```

## Use case using Either to handle errors
```typescript
// src/application/use-cases/create-team.use-case.ts
@Injectable()
export class CreateTeamUseCase {

    constructor(
        private readonly teamRepository: TeamRepository,
        private readonly transactionProvider: TransactionProvider,
        private readonly domainEventPublisher: DomainEventPublisher,
        private readonly idGenerator: IdGenerator,
    ) {}

    async execute(teamName: string): Promise<Either<CreateTeamError, string>> {
        const newTeam = Team.create(this.idGenerator.generate(), teamName); // Either<CreateTeamError, Team>
        if (isLeft(newTeam)) return newTeam; // early return instead of chains of map and flatMap

        await this.transactionProvider.run(async (trx) => {
            await this.teamRepository.save(newTeam.value, trx);
            await this.domainEventPublisher.publish(new TeamCreated(newTeam.value), trx);
        });

        return right(newTeam.value.id);
    }
}
```

## TypeScript specifics
- Expose the single entry point as `execute()`; name classes `<Action>UseCase` in kebab-case files (`create-team.use-case.ts`, `add-team-member.use-case.ts`) under `application/use-cases/`
- Express transactions through a generic port, `TransactionProvider.run<T>(work: (trx: Transaction) => Promise<T>): Promise<T>`, so the callback keeps its own return type instead of collapsing to `unknown`. `Transaction` stays deliberately empty - it is an opaque handle; adapters narrow it to their driver's type (see `building-blocks:implementing-repositories`), the use case only passes it on
- Type a command as an `interface` unless runtime decorators require a class. Nothing validates it at this layer, so a class buys nothing over a plain shape
- Return `Promise<void>` when the action has no result. Returning the saved entity "just in case" invites callers to depend on it
- TypeScript has no checked exceptions and no `either {}` builder; throw typed domain errors (see `cross-cutting:handling-errors-with-exceptions`), or, when the project uses `Either`, short-circuit with `if (isLeft(x)) return x;` after each step (see `cross-cutting:handling-errors-with-either`) rather than chaining result combinators
- A returned `Left` does not roll back a transaction - validate before opening it (as above), or throw inside it

## NestJS specifics
- Use `@Injectable()` for DI and inject repositories and external services (as abstract-class ports) via constructor
- Keep use cases framework-agnostic beyond DI: no decorators for HTTP, Kafka or other concerns
- Inject non-deterministic collaborators as ports (`IdGenerator`, `Clock`) - Nest DI cannot honour default constructor parameters
- Bind the `TransactionProvider` port to its adapter in the module, never construct it here - see `cross-cutting:configuring-runtime-dependencies`
