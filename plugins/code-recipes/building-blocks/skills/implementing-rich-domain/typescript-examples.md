# TypeScript Examples

## Aggregate root with an action verb
```typescript
// src/domain/team.domain.ts
export class Team {
    private static readonly MAX_MEMBERS = 10;

    private constructor(
        public readonly id: string,
        public readonly name: TeamName,
        public readonly members: ReadonlyArray<Member>,
        public readonly version: number,
    ) {}

    static create(id: string, name: string): Team {
        return new Team(id, TeamName.create(name), [], 0);
    }

    static reconstitute(id: string, name: string, members: ReadonlyArray<Member>, version: number): Team {
        return new Team(id, TeamName.reconstitute(name), members, version);
    }

    // A state change carries the version it was loaded from: the repository advances it on write and
    // rejects the write when that version is no longer the stored one (optimistic locking).
    addMember(member: Member): Team {
        if (this.members.length >= Team.MAX_MEMBERS) {
            throw new AddMemberException(AddMemberErrorType.MEMBER_LIMIT_REACHED, `Team ${this.id} has reached the maximum of ${Team.MAX_MEMBERS} members`);
        }
        if (this.members.some((existing) => existing.personId === member.personId)) {
            throw new AddMemberException(AddMemberErrorType.DUPLICATE_MEMBER, `Person ${member.personId} is already a member of team ${this.id}`);
        }
        return new Team(this.id, this.name, [...this.members, member], this.version);
    }

    canAcceptNewMember(): boolean {
        return this.members.length < Team.MAX_MEMBERS;
    }

    availableSpots(): number {
        return Team.MAX_MEMBERS - this.members.length;
    }

    equals(other: Team): boolean {
        return other.id === this.id;
    }
}

export interface Member {
    readonly personId: string;
}
```

## Value object with validated creation
```typescript
// src/domain/team.domain.ts
export class TeamName {
    private static readonly MAX_LENGTH = 255;

    private constructor(public readonly value: string) {}

    static create(value: string): TeamName {
        if (value.trim() === '') throw new TeamValidationException(TeamValidationErrorType.BLANK_NAME);
        if (value.length > TeamName.MAX_LENGTH) throw new TeamValidationException(TeamValidationErrorType.TOO_LONG_NAME);
        return new TeamName(value);
    }

    static reconstitute(value: string): TeamName {
        return new TeamName(value);
    }
}
```

## Domain errors
```typescript
// src/domain/team.errors.ts
export abstract class DomainException extends Error {
    protected constructor(message: string) {
        super(message);
        this.name = new.target.name;
    }
}

export class TeamNotFoundException extends DomainException {
    constructor(teamId: string) {
        super(`Team with id ${teamId} not found`);
    }
}

export enum AddMemberErrorType {
    MEMBER_LIMIT_REACHED = 'MEMBER_LIMIT_REACHED',
    DUPLICATE_MEMBER = 'DUPLICATE_MEMBER',
}

export class AddMemberException extends DomainException {
    constructor(public readonly type: AddMemberErrorType, message: string) {
        super(message);
    }
}

export enum TeamValidationErrorType {
    BLANK_NAME = 'BLANK_NAME',
    TOO_LONG_NAME = 'TOO_LONG_NAME',
}

export class TeamValidationException extends DomainException {
    constructor(public readonly errorType: TeamValidationErrorType) {
        super(`Invalid team: ${errorType}`);
    }
}
```

## Domain events
```typescript
// src/domain/team.events.ts
// Closed set of team events: a `switch` over `TeamEvent` is exhaustive
export type TeamEvent = TeamCreated | MemberAdded;

export class TeamCreated {
    readonly teamId: string;

    constructor(public readonly team: Team) {
        this.teamId = team.id;
    }
}

export class MemberAdded {
    constructor(public readonly teamId: string, public readonly personId: string) {}
}
```

## Use case acting on the new instance
```typescript
// src/application/use-cases/add-team-member.use-case.ts
@Injectable()
export class AddTeamMemberUseCase {

    constructor(
        private readonly teamRepository: TeamRepository,
        private readonly transactionProvider: TransactionProvider,
        private readonly domainEventPublisher: DomainEventPublisher,
    ) {}

    async execute(teamId: string, personId: string): Promise<void> {
        await this.transactionProvider.run(async (trx) => {
            const team = await this.teamRepository.find(teamId);
            if (!team) throw new TeamNotFoundException(teamId);

            const updated = team.addMember({personId});

            await this.teamRepository.save(updated, trx);
            await this.domainEventPublisher.publish(new MemberAdded(teamId, personId), trx);
        });
    }
}
```

## TypeScript specifics
- Mark every field `readonly` (collections as `ReadonlyArray`) and return a new instance from state changes (`[...this.members, member]`). Immutability then holds by construction rather than by reviewer vigilance
- Model a lifecycle state as a string-literal union (`'active' | 'archived'`), not an enum or bare `string`. The compiler enforces exhaustiveness and the valid values are readable at the declaration. Model closed event sets the same way, as a union of classes (`TeamEvent = TeamCreated | MemberAdded`)
- When creation must validate, make the constructor `private` and expose named static factories (`Team.create`, `TeamName.create`). The type system then makes the invalid path unreachable, instead of merely discouraged
- Keep creation and rehydration as separate named entry points (`create` / `reconstitute`). Data already in the database has passed validation once; re-running it on every read means a rule change retroactively makes stored rows unloadable
- TypeScript has no `equals`/`hashCode` contract: expose an explicit `equals(other)` comparing ids on entities, and never compare entities with `===`
- Set `this.name = new.target.name` in the base `DomainException` so every subclass reports its own name in logs and error responses
- Group types per concern in one file (`team.domain.ts`, `team.errors.ts`, `team.events.ts`) under `src/domain/`

## NestJS specifics
None - deliberately. The domain layer carries no `@Injectable()`, no decorators, and no imports from `@nestjs/*`, Kysely or KafkaJS. It is instantiated by use cases and adapters, never by the DI container, which is what keeps it testable without a framework and portable if the framework changes. Wiring belongs in `cross-cutting:configuring-runtime-dependencies`.
