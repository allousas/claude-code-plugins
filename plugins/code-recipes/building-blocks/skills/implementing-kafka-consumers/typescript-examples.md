# TypeScript Examples

All based on KafkaJS with NestJS DI. If the project uses NestJS microservices (`@EventPattern`) or another client, keep the same structure.

## KafkaJS Consumer
```typescript
// src/infrastructure/inbound/team-events.kafka-consumer.ts
@Injectable()
export class TeamEventsKafkaConsumer implements OnApplicationBootstrap {
    private readonly logger = new Logger(TeamEventsKafkaConsumer.name);

    constructor(
        private readonly kafkaConsumers: KafkaConsumerRunner,
        private readonly processTeamEvent: ProcessTeamEventUseCase,
    ) {}

    onApplicationBootstrap(): Promise<void> {
        return this.kafkaConsumers.run({topic: 'team-events', groupId: 'team-processor'}, (message) => this.consume(message));
    }

    async consume(message: KafkaMessage): Promise<void> {
        try {
            const event = JSON.parse(message.value?.toString() ?? '') as TeamEventMessage;
            if (!SUPPORTED_EVENT_TYPES.has(event.eventType)) {
                this.logger.log(`Ignoring unsupported event type: ${event.eventType}`); // do NOT throw → no retry, no DLQ
                return;
            }
            await this.processTeamEvent.execute(event.teamId, event.eventType);
            this.logger.log(`Processed team event: ${event.eventId}`);
        } catch (error) {
            this.logger.error('Failed to process team event', error instanceof Error ? error.stack : String(error));
            throw error; // let the runner retry and route to DLQ
        }
    }
}

const SUPPORTED_EVENT_TYPES: ReadonlySet<string> = new Set(['TEAM_CREATED', 'TEAM_ARCHIVED']);

interface TeamEventMessage {
    eventId: string;
    teamId: string;
    eventType: string;
}
```

## KafkaJS consumer replicating data from another topic
```typescript
// src/infrastructure/inbound/account-replication.kafka-consumer.ts
@Injectable()
export class AccountReplicationKafkaConsumer implements OnApplicationBootstrap {

    constructor(
        private readonly kafkaConsumers: KafkaConsumerRunner,
        private readonly repository: AccountProjectionRepository,
    ) {}

    onApplicationBootstrap(): Promise<void> {
        return this.kafkaConsumers.run({topic: 'account-events', groupId: 'account-replication'}, (message) => this.consume(message));
    }

    async consume(message: KafkaMessage): Promise<void> {
        const event = JSON.parse(message.value?.toString() ?? '') as AccountEventMessage;
        await this.repository.save(toProjection(event)); // access repository right away
    }
}
```

## Shared retry and DLQ configuration
```typescript
// src/infrastructure/config/kafka.consumer-runner.ts
const MAX_ATTEMPTS = 5;
const INITIAL_BACKOFF_MS = 500;

@Injectable()
export class KafkaConsumerRunner implements OnModuleInit, OnApplicationShutdown {
    private readonly dlqProducer: Producer;
    private consumers: ReadonlyArray<Consumer> = [];

    constructor(private readonly kafka: Kafka) {
        this.dlqProducer = kafka.producer();
    }

    async onModuleInit(): Promise<void> {
        await this.dlqProducer.connect();
    }

    async run(subscription: KafkaSubscription, handle: (message: KafkaMessage) => Promise<void>): Promise<void> {
        const consumer = this.kafka.consumer({groupId: subscription.groupId});
        this.consumers = [...this.consumers, consumer];
        await consumer.connect();
        await consumer.subscribe({topic: subscription.topic});
        await consumer.run({
            eachMessage: async ({topic, message}) => {
                try {
                    await withExponentialBackoff(() => handle(message));
                } catch (error) {
                    await this.dlqProducer.send({
                        topic: `${topic}.DLT`,
                        messages: [{key: message.key, value: message.value, headers: {...message.headers, 'x-exception': String(error)}}],
                    });
                }
            },
        });
    }

    async onApplicationShutdown(): Promise<void> {
        await Promise.all(this.consumers.map((consumer) => consumer.disconnect()));
        await this.dlqProducer.disconnect();
    }
}

async function withExponentialBackoff(work: () => Promise<void>, attempt = 1): Promise<void> {
    try {
        await work();
    } catch (error) {
        if (attempt >= MAX_ATTEMPTS) throw error;
        await new Promise((resolve) => setTimeout(resolve, INITIAL_BACKOFF_MS * 2 ** (attempt - 1)));
        await withExponentialBackoff(work, attempt + 1);
    }
}

export interface KafkaSubscription {
    topic: string;
    groupId: string;
}
```

## TypeScript / NestJS specifics
- KafkaJS has no built-in DLQ: when `eachMessage` throws it retries and eventually crashes the consumer. Centralise retries with backoff and dead-lettering in one config component (`KafkaConsumerRunner`) so individual consumers only parse, delegate and rethrow
- Start consumers in `onApplicationBootstrap` (after every provider's `onModuleInit`) and disconnect them in `onApplicationShutdown`; requires `app.enableShutdownHooks()` in `main.ts`
- Provide the `Kafka` client (brokers, clientId) from a config provider - see `cross-cutting:configuring-runtime-dependencies`
- Type message payloads as `interface`s declared below the consumer and unexported; parse with `JSON.parse` (or a schema validator such as zod when the producer is not trusted) before mapping to primitives
- Name files `<topic>.kafka-consumer.ts`
