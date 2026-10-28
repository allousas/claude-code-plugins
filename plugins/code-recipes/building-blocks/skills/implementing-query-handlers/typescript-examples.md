# TypeScript Examples

## Query Handler
```typescript
// src/application/queries/can-team-accept-member.query-handler.ts
@Injectable()
export class CanTeamAcceptMemberQueryHandler {

    constructor(private readonly teamRepository: TeamRepository) {}

    async execute(query: CanTeamAcceptMemberQuery): Promise<TeamCapacityView> {
        const team = await this.teamRepository.find(query.teamId);
        if (!team) throw new TeamNotFoundException(query.teamId);

        return {teamId: team.id, canAcceptNewMember: team.canAcceptNewMember(), availableSpots: team.availableSpots()};
    }
}

export interface CanTeamAcceptMemberQuery {
    readonly teamId: string;
}

export interface TeamCapacityView {
    readonly teamId: string;
    readonly canAcceptNewMember: boolean;
    readonly availableSpots: number;
}
```

## Wiring to a controller
```typescript
@Get(':teamId/capacity')
capacity(@Param('teamId', ParseUUIDPipe) teamId: string): Promise<TeamCapacityView> {
    return this.canTeamAcceptMember.execute({teamId});
}
```

## TypeScript specifics
- Declare the query and projection types in the same file as the handler, below the class, as `interface`s - the shape belongs to the query, and a reader should not have to open a second file to learn what comes back
- Mark projection fields `readonly` and type collection results as `ReadonlyArray<T>`. A projection is a snapshot, and `readonly` says so at the type level instead of by convention
- Give the projection an explicit type rather than inferring from a row shape or passing `any` through. When the repository's shape changes, the mismatch should surface as a compile error here rather than as a silently wrong response
- Expose the entry point as `execute()`; name files `<query>.query-handler.ts` under `application/queries/`

## NestJS specifics
- Use `@Injectable()` for DI and inject repositories (as abstract-class ports) via constructor
- Keep query handlers framework-agnostic beyond DI
- Wire them to `@Get()` handlers in controllers
