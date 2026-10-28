# TypeScript Examples

All based on Kysely + PostgreSQL with NestJS DI.

## Kysely Database Repository
```typescript
// src/infrastructure/outbound/postgres.team-repository.ts
type TeamRow = Selectable<Database['teams']>;

@Injectable()
export class PostgresTeamRepository implements TeamRepository {

    constructor(private readonly db: Kysely<Database>) {}

    async find(teamId: string): Promise<Team | null> {
        const row = await this.db
            .selectFrom('teams')
            .selectAll()
            .where('id', '=', teamId)
            .executeTakeFirst();

        return row ? this.toTeam(row) : null;
    }

    async save(team: Team, transaction: Transaction): Promise<void> {
        const trx = activeTransaction(transaction, `Saving team ${team.id}`);

        const result = await trx
            .insertInto('teams')
            .values({id: team.id, name: team.name.value, member_ids: team.members.map((member) => member.personId), version: 0})
            .onConflict((oc) => oc
                .column('id')
                .doUpdateSet((eb) => ({
                    name: eb.ref('excluded.name'),
                    member_ids: eb.ref('excluded.member_ids'),
                    version: sql<number>`teams.version + 1`,
                    updated_at: sql<Date>`now()`,
                }))
                .where('teams.version', '=', team.version))
            .executeTakeFirst();

        if (result.numInsertedOrUpdatedRows === 0n) throw new OptimisticLockingException('Team', team.id);
    }

    private toTeam(row: TeamRow): Team {
        return Team.reconstitute(row.id, row.name, row.member_ids.map((personId) => ({personId})), row.version);
    }
}
```

## Schema types and transaction handle
```typescript
// src/infrastructure/outbound/database.schema.ts
interface TeamsTable {
    id: string;
    name: string;
    member_ids: string[];
    version: number;
    created_at: Generated<Date>;
    updated_at: Generated<Date>;
}

export interface Database {
    teams: TeamsTable;
}

export interface KyselyTransaction extends Transaction {
    readonly trx: Kysely<Database>;
}

export function activeTransaction(transaction: Transaction, operation: string): Kysely<Database> {
    const {trx} = (transaction ?? {}) as Partial<KyselyTransaction>;

    if (trx?.isTransaction !== true) throw new TransactionNotActiveException(operation);

    return trx;
}
```

## TypeScript specifics
- Return `Promise<Entity | null>` for a lookup that can miss. The absence is then part of the signature, so the caller is forced to handle it rather than discovering `undefined` at runtime
- Derive row types from the schema (`Selectable<Database['teams']>`) rather than hand-writing them, and leave them unexported so the "never leak DB DTOs out" rule is enforced by the module boundary rather than by review. A migration that changes a column then breaks compilation here, which is exactly where you want to find out
- Rehydrate through the domain's `reconstitute` factory, never `create` - stored rows have already been validated once
- Narrow the domain's opaque `Transaction` port to the driver's handle at this boundary (`KyselyTransaction`), so the domain stays driver-free while the adapter stays typed. Validate the handle before use rather than trusting the cast - an inactive or absent transaction should fail with a named infrastructure exception (`TransactionNotActiveException`), not a `TypeError` deep in the driver
- Write methods take the `Transaction` as a parameter; the use case opens it through `TransactionProvider` (see `building-blocks:implementing-application-services`)
- Name files `<storage>.<entity>-repository.ts` (`postgres.team-repository.ts`)

## NestJS specifics
- Use `@Injectable()` for DI
- Bind the repository to its domain port in the module (`{provide: TeamRepository, useClass: PostgresTeamRepository}`) - see `cross-cutting:configuring-runtime-dependencies`
