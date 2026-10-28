---
name: implementing-kafka-producers
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing Kafka producers or any component that publishes events/messages to Kafka (e.g. Spring `KafkaTemplate`, KafkaJS `producer.send`).
---

## Purpose
Kafka producers that publish events/messages to kafka broker.

## Typical Flow
1. Receive domain event or message
2. Serialize domain event/object to message format (JSON, protobuf, Avro, etc.)
3. Publish to Kafka topic

## Guidelines

**DO:**
- Serialize domain events/objects to message format, mapping to a message DTO first (never serialize domain objects as they are)
- Use the aggregate id as message key to keep per-entity ordering
- Handle producer errors appropriately (log, retry, circuit breaker)
- File naming: `Kafka<Concern>Producer`, e.g., `KafkaTeamEventProducer`, if an interface is implemented, then follow interface naming convention if any
- Declare message DTOs in the same file for cohesion
- Implement a domain/outbound port, so application services never depend on Kafka
- For reliable publishing tied to a database change, see `patterns:applying-outbox-pattern`

**DON'T:**
- Include business logic
- Throw domain exceptions or return domain errors - use infrastructure exceptions/errors if needed

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
