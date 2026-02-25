---
name: applying-outbox-pattern
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing any code that implements the transactional outbox pattern. This includes adding, modifying any outbox related files/classes but also when performing dual writes in the code base, specially in application service layer.
---

## Purpose
Guarantee that domain events are published reliably by storing them in an outbox table within the same database transaction as the state change. 
A separate process reads the outbox and publishes events to the message broker, ensuring at-least-once delivery and **avoiding the dual-write problem**.

## Typical Flow
1. Application service performs domain operation and persists state change
2. In the same transaction, application service writes event to the outbox table via `OutboxRepository`
   - The event is an integration event (e.g. protobuf or JSON message ...), not a domain event object
3. Transaction commits atomically (state change + outbox entry)
4. A separate poller or CDC process reads unpublished outbox entries
5. Poller publishes events to Kafka (or other broker)
6. Poller marks outbox entries as published
7. Consumers handle events idempotently (at-least-once delivery means duplicates are possible)

## Guidelines

**DO:**
- Make sure that the application service or code that triggered the upper layer action os wrapped in a transaction
- Store events in the outbox table within the same database transaction as the entity state change
- Include `id`, `aggregate_id`, `event_type`, `payload` (JSON), `created_at`, and `published` flag in the outbox table
- Use a separate scheduled poller or CDC (Change Data Capture) to read and publish events
- Mark events as published after successful broker delivery
- Combine with the  `DomainEventPublisher` pattern to publish domain events 

**DON'T:**
- Publish to Kafka and write to DB in separate transactions (dual-write problem - one can succeed while the other fails)
- Delete outbox entries immediately after publishing - keep them for debugging/auditing and clean up with a scheduled job
- Process outbox entries without ordering guarantees - use `created_at` ordering per aggregate
- Skip idempotency in consumers - at-least-once delivery means duplicates will happen

### Spring specifics
- Use `@Scheduled` for the outbox poller
- The poller runs its own `@Transactional` to read and mark entries
- Use `@EnableScheduling` in a configuration class

## Examples
Please use always these examples as reference: [examples.md](examples.md)
