# TypeScript Examples

## Outbox table schema
```sql
-- db/migrations/20240101000000_create_outbox.sql (dbmate)
-- migrate:up
CREATE TABLE outbox (
    id           UUID         PRIMARY KEY,
    aggregate_id UUID         NOT NULL,
    event_type   VARCHAR(255) NOT NULL,
    payload      JSONB        NOT NULL,
    created_at   TIMESTAMPTZ  NOT NULL DEFAULT now(),
    published    BOOLEAN      NOT NULL DEFAULT false
);

CREATE INDEX idx_outbox_unpublished ON outbox (created_at) WHERE published = false;

-- migrate:down
DROP TABLE outbox;
```

## OutboxRepository
```typescript
// infrastructure/outbound/postgres.outbox-repository.ts
@Injectable()
export class PostgresOutboxRepository {
  async store(entry: OutboxEntry, transaction: Transaction): Promise<void> {
    const trx = activeTransaction(transaction, `Storing outbox entry ${entry.id}`);

    await trx
      .insertInto('outbox')
      .values({
        id: entry.id,
        aggregate_id: entry.aggregateId,
        event_type: entry.eventType,
        payload: entry.payload,
        created_at: entry.createdAt,
      })
      .execute();
  }

  async findUnpublished(transaction: Transaction, limit = 100): Promise<ReadonlyArray<OutboxEntry>> {
    const trx = activeTransaction(transaction, 'Reading unpublished outbox entries');

    const rows = await trx
      .selectFrom('outbox')
      .select(['id', 'aggregate_id', 'event_type', 'payload', 'created_at'])
      .where('published', '=', false)
      .orderBy('created_at')
      .limit(limit)
      .forUpdate()
      .skipLocked()
      .execute();

    return rows.map((row) => this.toEntry(row));
  }

  async markAsPublished(ids: ReadonlyArray<string>, transaction: Transaction): Promise<void> {
    const trx = activeTransaction(transaction, 'Marking outbox entries as published');

    await trx.updateTable('outbox').set({ published: true }).where('id', 'in', ids).execute();
  }

  private toEntry(row: OutboxRow): OutboxEntry {
    return {
      id: row.id,
      aggregateId: row.aggregate_id,
      eventType: row.event_type,
      payload: JSON.stringify(row.payload),
      createdAt: row.created_at,
    };
  }
}

type OutboxRow = Pick<Selectable<Database['outbox']>, 'id' | 'aggregate_id' | 'event_type' | 'payload' | 'created_at'>;

export interface OutboxEntry {
  readonly id: string;
  readonly aggregateId: string;
  readonly eventType: string;
  readonly payload: string;
  readonly createdAt: Date;
}
```

## Outbox-based DomainEventPublisher adapter
```typescript
// infrastructure/outbound/outbox.domain-event-publisher.ts
@Injectable()
export class OutboxDomainEventPublisher implements DomainEventPublisher {
  constructor(private readonly outboxRepository: PostgresOutboxRepository) {}

  async publish(event: DomainEvent, transaction: Transaction): Promise<void> {
    const integrationEvent = toIntegrationEvent(event); // this could be protobuf, json, avro ...
    await this.outboxRepository.store(
      {
        id: event.eventId,
        aggregateId: event.teamId,
        eventType: event.constructor.name,
        payload: JSON.stringify(integrationEvent),
        createdAt: event.occurredAt,
      },
      transaction,
    );
  }
} // if domain-event-publisher pattern with multiple in process subscribers is used, this class could be a subscriber of DomainEvent

// integration event: only what consumers need
const toIntegrationEvent = (event: DomainEvent) => ({ eventId: event.eventId, teamId: event.teamId, eventType: event.constructor.name });
```

## Use case (unchanged - uses the DomainEventPublisher port)
```typescript
// application/use-cases/create-team.use-case.ts
@Injectable()
export class CreateTeamUseCase {
  constructor(
    private readonly teamRepository: TeamRepository,
    private readonly transactionProvider: TransactionProvider,
    private readonly domainEventPublisher: DomainEventPublisher,
    private readonly idGenerator: IdGenerator,
  ) {}

  async execute(teamName: string): Promise<string> {
    return this.transactionProvider.run(async (trx) => {
      const newTeam = Team.create(this.idGenerator.generate(), teamName);
      await this.teamRepository.save(newTeam, trx);
      await this.domainEventPublisher.publish(new TeamCreated(newTeam.id, teamName), trx);
      return newTeam.id;
    });
  }
}
```

## Outbox poller publishing to Kafka
```typescript
// infrastructure/inbound/outbox.poller.ts
@Injectable()
export class OutboxPoller implements OnApplicationBootstrap, OnModuleDestroy {
  private readonly logger = new Logger(OutboxPoller.name);
  private timer?: NodeJS.Timeout;

  constructor(
    private readonly transactionProvider: TransactionProvider,
    private readonly outboxRepository: PostgresOutboxRepository,
    @Inject(KAFKA_PRODUCER) private readonly producer: Producer,
  ) {}

  onApplicationBootstrap(): void {
    this.schedule();
  }

  onModuleDestroy(): void {
    clearTimeout(this.timer);
    this.timer = undefined;
  }

  async poll(): Promise<void> {
    await this.transactionProvider.run(async (trx) => {
      const entries = await this.outboxRepository.findUnpublished(trx);
      if (entries.length === 0) return;

      // resolves on the broker ack
      await this.producer.send({
        topic: TOPIC,
        messages: entries.map((entry) => ({ key: entry.aggregateId, value: entry.payload, headers: { 'event-type': entry.eventType } })),
      });
      await this.outboxRepository.markAsPublished(entries.map((entry) => entry.id), trx);
      this.logger.log(`Published ${entries.length} outbox entries`);
    });
  }

  // chained timeouts, so a slow poll never overlaps the next one
  private schedule(): void {
    this.timer = setTimeout(async () => {
      await this.poll().catch((error) => this.logger.error('Outbox poll failed', error));
      if (this.timer) this.schedule();
    }, POLL_INTERVAL_MS);
  }
}

const TOPIC = 'domain-events';
const POLL_INTERVAL_MS = 100;
```

## Schema types (Kysely)
```typescript
// infrastructure/outbound/database.schema.ts
interface OutboxTable {
  id: string;
  aggregate_id: string;
  event_type: string;
  payload: ColumnType<object, string, never>; // read parsed jsonb, write JSON text
  created_at: Generated<Date>;
  published: Generated<boolean>;
}

export interface Database {
  teams: TeamsTable;
  outbox: OutboxTable;
}
```

## Wiring
```typescript
// infrastructure/config/kafka.providers.ts
export const KAFKA_PRODUCER = 'KAFKA_PRODUCER';

export const kafkaProducerProvider: Provider = {
  provide: KAFKA_PRODUCER,
  useFactory: async (): Promise<Producer> => {
    const producer = new Kafka({ clientId: 'team-service', brokers: [process.env.KAFKA_BROKERS ?? 'localhost:9092'] })
      .producer({ idempotent: true });
    await producer.connect();
    return producer;
  },
};

// infrastructure/config/app.module.ts
@Module({
  providers: [
    CreateTeamUseCase,
    PostgresOutboxRepository,
    OutboxPoller,
    kafkaProducerProvider,
    { provide: DomainEventPublisher, useClass: OutboxDomainEventPublisher },
  ],
})
export class AppModule {}
```

## TypeScript / NestJS specifics
- Transactions are explicit: the `DomainEventPublisher` port already carries the `Transaction`; the `OutboxDomainEventPublisher` (or the in-memory publisher, when the outbox is one of its subscribers) forwards it to the outbox repository, which narrows it to the Kysely handle with `activeTransaction` (fails with a named infrastructure exception when absent)
- The outbox repository and publisher are infrastructure-to-infrastructure collaborators — no domain port needed for the repository
- Run the poller from NestJS lifecycle hooks (`OnApplicationBootstrap` / `OnModuleDestroy`) with chained `setTimeout`s, or `@Interval` from `@nestjs/schedule` if that dependency is already present; each poll opens its own transaction through `TransactionProvider`
- `await producer.send(...)` resolves on broker acknowledgement — only then mark entries as published
- Third-party interfaces such as the KafkaJS `Producer` erase at runtime, so they need an explicit provider token (see `cross-cutting:configuring-runtime-dependencies`)
- Integration tests for the outbox need real commits/rollbacks against Postgres — don't use rolled-back-transaction test isolation here
