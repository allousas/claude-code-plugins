# TypeScript Examples

All based on KafkaJS with NestJS DI.

## KafkaJS producer for an integration message
```typescript
// src/infrastructure/outbound/kafka.team-event-producer.ts
const TOPIC = 'team.events';

@Injectable()
export class KafkaTeamEventProducer implements TeamEventPublisher, OnModuleInit, OnApplicationShutdown {
    private readonly producer: Producer;

    constructor(kafka: Kafka) {
        this.producer = kafka.producer({idempotent: true});
    }

    async onModuleInit(): Promise<void> {
        await this.producer.connect();
    }

    async publish(event: TeamEvent): Promise<void> {
        try {
            await this.producer.send({
                topic: TOPIC,
                messages: [{key: event.teamId, value: JSON.stringify(this.toMessage(event))}],
            });
        } catch (error) {
            throw new KafkaPublishException(TOPIC, error);
        }
    }

    async onApplicationShutdown(): Promise<void> {
        await this.producer.disconnect();
    }

    private toMessage(event: TeamEvent): TeamEventMessage {
        return {teamId: event.teamId, eventType: event.constructor.name, occurredAt: new Date().toISOString()};
    }
}

interface TeamEventMessage {
    teamId: string;
    eventType: string;
    occurredAt: string;
}
```

```typescript
// src/infrastructure/infrastructure.errors.ts
export class KafkaPublishException extends InfrastructureException {
    constructor(topic: string, cause?: unknown) {
        super(`Failed to publish to ${topic}`, cause);
    }
}
```

## TypeScript / NestJS specifics
- Use the KafkaJS `Producer` (`producer.send`); connect it in `onModuleInit` and disconnect in `onApplicationShutdown` (requires `app.enableShutdownHooks()`)
- Serialize with `JSON.stringify` over a message `interface` declared below the class, never the domain object itself - class instances serialize their internals, not a contract
- KafkaJS already retries transient broker errors (`retry` option on the client); wrap the final failure in an infrastructure exception rather than a domain error
- Provide the `Kafka` client from a config provider and bind the producer to its port with `{provide: TeamEventPublisher, useClass: KafkaTeamEventProducer}` - see `cross-cutting:configuring-runtime-dependencies`
- Name files `kafka.<concern>-producer.ts`
