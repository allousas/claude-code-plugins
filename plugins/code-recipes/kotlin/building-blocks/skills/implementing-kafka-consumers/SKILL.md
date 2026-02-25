---
name: implementing-kafka-consumers
description: Apply when creating, refactoring, modifying, planning (plan mode) or reviewing any file that is a Kafka consumer, kafka message listener, or kafka event consumer. 
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
- Delegate to global configured mechanisms for retries and recovery
- File naming: `<Topic><StreamSystem>Consumer`, e.g., `OrdersKafkaConsumer`
- Declare message DTOs in the same file for cohesion
- Could call repositories directly for idempotency checks
- Could call repositories directly for data replication use cases

**DON'T:**
- Include business logic - only message parsing, delegation, and error handling
- Suppress exceptions silently - rethrow after logging to allow retry/DLQ mechanisms
- Do not leak kafka specific dtos or classes to application services

### Spring specifics
- Use `@KafkaListener` annotation
- Ensure `@EnableKafka` is enabled
- Use `KafkaConfig` class for separate configuration
- Ensure `DefaultErrorHandler` with exponential backoff is configured through `ConcurrentKafkaListenerContainerFactory`
- Ensure `DeadLetterPublishingRecoverer` is configured

## Examples
Please use always these examples as reference: [examples.md](examples.md)