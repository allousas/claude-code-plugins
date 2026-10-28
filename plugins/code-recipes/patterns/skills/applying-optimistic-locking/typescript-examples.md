# TypeScript Examples

## Entity with version field
```typescript
// domain/team.domain.ts
export class Team {
  constructor(
    public readonly id: string,
    public readonly name: string,
    public readonly version: number,
  ) {}

  static create(id: string, name: string): Team {
    return new Team(id, name, 0);
  }

  // the new state carries the version it was loaded from; the repository advances it on write
  rename(newName: string): Team {
    return new Team(this.id, newName, this.version);
  }
}
```

## OptimisticLockingException
```typescript
// infrastructure/infrastructure.errors.ts
export abstract class InfrastructureException extends Error {
  protected constructor(message: string, public readonly cause?: unknown) {
    super(message);
    this.name = new.target.name;
  }
}

export class OptimisticLockingException extends InfrastructureException {
  constructor(entityId: string, cause?: unknown) {
    super(`Concurrent modification detected for entity ${entityId}`, cause);
  }
}
```

## Repository with version check
```typescript
// infrastructure/outbound/postgres.team-repository.ts
@Injectable()
export class PostgresTeamRepository implements TeamRepository {
  async save(team: Team, transaction: Transaction): Promise<void> {
    const trx = activeTransaction(transaction, `Saving team ${team.id}`);

    const result = await trx
      .insertInto('teams')
      .values({ id: team.id, name: team.name, version: 0 })
      .onConflict((oc) =>
        oc
          .column('id')
          .doUpdateSet((eb) => ({
            name: eb.ref('excluded.name'),
            version: sql<number>`teams.version + 1`,
          }))
          .where('teams.version', '=', team.version),
      )
      .executeTakeFirst();

    if (!result.numInsertedOrUpdatedRows) throw new OptimisticLockingException(team.id);
  }
}
```

## TypeScript / NestJS specifics
- Kysely returns affected counts as `bigint` (`numUpdatedRows`, `numInsertedOrUpdatedRows`) — compare against `0n` or test falsiness, never `=== 0`
- Use a `sql` template (or `eb('version', '+', 1)`) so the increment happens in the statement, not in TypeScript
- When the repository only inserts (no upsert), catch the Postgres unique violation (`23505`) and rethrow it as `OptimisticLockingException` with the driver error as `cause`; let any other driver error propagate
- Integration tests isolate with a truncate before each test, not rolled-back transactions — the version check needs genuine commits
