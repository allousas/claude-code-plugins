# Examples

## Sealed error type for a use case
```typescript
// domain/team.errors.ts
export type CreateTeamErrorType = 'INVALID_TEAM_NAME'; // other error types could be added here

export interface CreateTeamError {
    readonly kind: 'CreateTeamError';
    readonly type: CreateTeamErrorType;
    readonly details?: string;
}

export type DomainError = CreateTeamError; // | AddMemberError | ...
```

## Domain function returning Either
```typescript
// domain/team.domain.ts
export class Team {
    private constructor(public readonly id: string, public readonly name: string) {}

    static create(id: string, name: string): Either<CreateTeamError, Team> {
        if (name.trim() === '') {
            return left({kind: 'CreateTeamError', type: 'INVALID_TEAM_NAME', details: 'Name cannot be blank'});
        }
        return right(new Team(id, name));
    }
}
```

## Use case using early return
```typescript
// application/use-cases/create-team.use-case.ts
@Injectable()
export class CreateTeamUseCase {
    constructor(
        private readonly teamRepository: TeamRepository,
        private readonly domainEventPublisher: DomainEventPublisher,
        private readonly transactionProvider: TransactionProvider,
        private readonly idGenerator: IdGenerator,
    ) {}

    async execute(teamName: string): Promise<Either<CreateTeamError, string>> {
        const newTeam = Team.create(this.idGenerator.generate(), teamName);
        if (isLeft(newTeam)) return newTeam;

        await this.transactionProvider.run(async (trx) => {
            await this.teamRepository.save(newTeam.value, trx);
            await this.domainEventPublisher.publish(new TeamCreated(newTeam.value), trx);
        });

        return right(newTeam.value.id);
    }
}
```

## Controller mapping Either to HTTP response
```typescript
// infrastructure/inbound/team.controller.ts
@Controller('teams')
export class TeamController {
    constructor(private readonly createTeam: CreateTeamUseCase) {}

    @Post()
    async create(
        @Body() request: CreateTeamRequest,
        @Res({passthrough: true}) response: Response,
    ): Promise<CreateTeamResponse | ErrorResponse> {
        const result = await this.createTeam.execute(request.name);

        if (isLeft(result)) {
            const error = toErrorResponse(result.value);
            response.status(error.status);
            return error.body;
        }

        response.status(HttpStatus.CREATED).location(`/teams/${result.value}`);
        return {id: result.value};
    }
}

function toErrorResponse(error: CreateTeamError): {status: HttpStatus; body: ErrorResponse} {
    switch (error.type) {
        case 'INVALID_TEAM_NAME':
            return {status: HttpStatus.UNPROCESSABLE_ENTITY, body: {code: 'INVALID_TEAM_NAME', message: error.details}};
        // other maps here
        default: {
            const unhandled: never = error.type; // compile error when a new error type is not mapped
            throw new Error(`Unmapped error ${unhandled}`);
        }
    }
}
```
```typescript
// infrastructure/inbound/team.http-dtos.ts
export interface CreateTeamRequest {
    readonly name: string;
}

export interface CreateTeamResponse {
    readonly id: string;
}

export interface ErrorResponse {
    readonly code: string;
    readonly message?: string;
}
```

## Minimal Either type (no library)
```typescript
// domain/either.ts
export type Either<L, R> = Left<L> | Right<R>;

export interface Left<L> {
    readonly kind: 'left';
    readonly value: L;
}

export interface Right<R> {
    readonly kind: 'right';
    readonly value: R;
}

export const left = <L>(value: L): Left<L> => ({kind: 'left', value});
export const right = <R>(value: R): Right<R> => ({kind: 'right', value});
export const isLeft = <L, R>(either: Either<L, R>): either is Left<L> => either.kind === 'left';
```

## TypeScript / NestJS specifics
- Prefer the small discriminated-union `Either` above over a library; reach for a library (e.g. `effect`, `neverthrow`) only if the project already uses one
- TypeScript has no `either {}` builder — short-circuit with `if (isLeft(x)) return x;` after each step instead of chaining `map`/`flatMap`; the type guard narrows `x.value` to the success type afterwards
- Use string-literal unions for error types (TypeScript's closest equivalent to a sealed type) and a `never` check in the `default` branch so unmapped variants fail compilation
- A returned `Left` does not roll back a transaction — validate before opening it, or throw inside it
- In NestJS controllers, map a `Left` to the HTTP status and error body explicitly (`@Res({passthrough: true})` to set the status and `Location`); the global exception filter still handles infrastructure exceptions
