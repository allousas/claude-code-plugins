# TypeScript Examples

## In-memory publisher and subscribers
```typescript
// domain/outbound.ports.ts
export interface Transaction {}

export abstract class DomainEventPublisher {
  abstract publish(event: DomainEvent, transaction: Transaction): Promise<void>;
}

// domain/team.events.ts
export class TeamCreated {
  constructor(public readonly team: Team) {}
}

export class TeamMemberJoined {
  constructor(public readonly team: Team, public readonly personId: string) {}
}

export class TeamMemberLeft {
  constructor(public readonly team: Team, public readonly personId: string) {}
}

export type DomainEvent = TeamCreated | TeamMemberJoined | TeamMemberLeft;

// infrastructure/outbound/in-memory.domain-event-publisher.ts
@Injectable()
export class InMemoryDomainEventPublisher implements DomainEventPublisher {
  constructor(
    private readonly logging: LoggingSubscriber,
    private readonly metrics: MetricsSubscriber,
  ) {}

  // `transaction` is accepted by the port; take it as a second parameter once a subscriber needs it (outbox)
  async publish(event: DomainEvent): Promise<void> {
    await this.logging.handle(event);
    await this.metrics.handle(event);
  }
}

// infrastructure/outbound/domain-event.subscribers.ts
@Injectable()
export class LoggingSubscriber {
  private readonly logger = new Logger(LoggingSubscriber.name);

  async handle(event: DomainEvent): Promise<void> {
    this.logger.log(logMessage(event));
  }
}

@Injectable()
export class MetricsSubscriber {
  private readonly counter = new Counter({ name: 'domain_event_total', help: 'Domain events published', labelNames: ['type'] });

  async handle(event: DomainEvent): Promise<void> {
    this.counter.inc({ type: event.constructor.name });
  }
}

// event type and identifiers only — never the whole event
const logMessage = (event: DomainEvent): string => {
  const commonMessage = `domain-event: '${event.constructor.name}', team-id: '${event.team.id}'`;
  const specificMessage = event instanceof TeamCreated ? '' : `, person-id: ${event.personId}`;
  return `${commonMessage}${specificMessage}`;
};
```

## Wiring
```typescript
// infrastructure/config/app.module.ts
@Module({
  providers: [
    CreateTeamUseCase,
    LoggingSubscriber,
    MetricsSubscriber,
    { provide: DomainEventPublisher, useClass: InMemoryDomainEventPublisher },
  ],
})
export class AppModule {}
```

## TypeScript / NestJS specifics
- Declare the publisher port as an `abstract class`, not an `interface` — it must survive compilation to act as a DI token (see `cross-cutting:configuring-runtime-dependencies`)
- Each event is a `readonly` class in the `DomainEvent` union, named in the past tense (`TeamCreated`)
- Dispatch with a sequential `for...of`/`await` per subscriber — never `Promise.all`, which puts one transaction handle into concurrent use the driver does not support
- Inject subscribers as ordinary providers, one constructor argument each — no `Symbol` token, no registry class, no subscriber interface
- Adding a subscriber means editing the publisher too (constructor argument + call in `publish`); nothing fails at boot if you forget, so add its dispatch assertion in the same change
- Bind the port to the adapter in the module (`{provide: DomainEventPublisher, useClass: InMemoryDomainEventPublisher}`)
- Metrics subscriber uses `prom-client` (add the dependency when the subscriber is introduced)
