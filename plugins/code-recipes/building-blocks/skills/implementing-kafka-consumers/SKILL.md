---
name: implementing-kafka-consumers
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing Kafka consumers or other message/stream consumers (e.g. `@KafkaListener` classes, KafkaJS consumers, NestJS `@EventPattern` handlers, classes named `*Consumer`/`*Listener`).
---

## Purpose
Message consumers listen to message queues (Kafka, RabbitMQ) and trigger application services. They translate external messages into application service calls.

## Typical Flow
1. Receive a message from Kafka topic
2. Parse and validate incoming message
3. Filter out messages that are not relevant for the current consumer
4. Transform message into application service parameters (DTOs or primitives)
5. Call application service to trigger business use case
6. Log and publish metrics for observability
7. On failure, log error and rethrow to let Kafka error handling mechanisms (retries, DLQ) work

## Guidelines

**DO:**
- Delegate to global configured mechanisms for retries (with exponential backoff) and recovery (dead letter topic)
- File naming: `<Topic><StreamSystem>Consumer`, e.g., `OrdersKafkaConsumer`
- Declare message DTOs in the same file for cohesion
- Could call repositories directly for idempotency checks
- Could call repositories directly for data replication use cases
- Keep consumer configuration (brokers, group ids, error handling) in a separate configuration component

**DON'T:**
- Include business logic - only message parsing, delegation, and error handling
- Suppress exceptions silently - rethrow after logging to allow retry/DLQ mechanisms
- Throw for irrelevant messages - ignore them (log and return) so they are not retried or dead-lettered
- Leak kafka specific dtos or classes to application services

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
