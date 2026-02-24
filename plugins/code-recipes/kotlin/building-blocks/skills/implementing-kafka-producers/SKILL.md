---
name: implementing-kafka-producers
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing Kafka producers.
---

## Purpose
Kafka producers that publish events/messages to kafka broker.

## Typical Flow
1. Receive domain event or message
2. Serialize domain event/object to message format (JSON, protobuf, Avro, etc.)
3. Publish to Kafka topic

## Guidelines

**DO:**
- Serialize domain events/objects to message format
- Handle producer errors appropriately (log, retry, circuit breaker)
- File naming: `Kafka<Concern>Producer`, e.g., `KafkaTeamEventProducer`, if an interface is implemented, then follow interface naming convention if any
- Declare message DTOs in the same file for cohesion

**DON'T:**
- Include business logic
- Throw domain exceptions or return domain errors - use infrastructure exceptions/errors if needed

### Spring specifics
- Use kafka template to publish messages
- Delegate serialization to `ObjectMapper`
- Use `@Component` annotation do declare the bean

## Examples
Please use always these examples as reference: [examples.md](examples.md)
