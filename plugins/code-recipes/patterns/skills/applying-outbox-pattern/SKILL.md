---
name: applying-outbox-pattern
description: Invoke when writing or reviewing outbox code (including plans), when a use case writes to the database and publishes to a broker in the same flow, or when the user asks about reliable event publishing or the dual-write problem. Covers outbox tables, outbox repositories, and pollers, relays or CDC to a broker.
---

## Purpose
Guarantee that domain events are published reliably by storing them in an outbox table within the same database transaction as the state change. 
A separate process reads the outbox and publishes events to the message broker, ensuring at-least-once delivery and **avoiding the dual-write problem**.

## When to introduce it
When a consumer outside the service needs the event (or the service becomes production-ready with cross-service events). Until then, in-process handlers are enough (`patterns:applying-domain-event-publisher`).

## Typical Flow
1. Application service performs domain operation and persists state change
2. In the same transaction, the event is written to the outbox table via an `OutboxRepository` (typically from an outbox handler of the `DomainEventPublisher`)
   - The stored event is an integration event (e.g. protobuf, JSON, Avro message), not the domain event object
3. Transaction commits atomically (state change + outbox entry); a rollback leaves no outbox row
4. A separate poller or CDC process reads unpublished outbox entries
5. Poller publishes events to Kafka (or other broker) and waits for the broker acknowledgement
6. Poller marks outbox entries as published
7. Consumers handle events idempotently (at-least-once delivery means duplicates are possible)

## Guidelines

**DO:**
- Make sure the application service (or the code that triggered the action) is wrapped in a transaction
- Store events in the outbox table within the same database transaction as the entity state change — pass the caller's transaction to the outbox write when transactions are explicit
- Include `id`, `aggregate_id`, `event_type`, `payload` (JSON), `created_at`, and `published` flag in the outbox table
- Use a separate scheduled poller or CDC (Change Data Capture) to read and publish events
- Lock the rows the poller reads (e.g. `FOR UPDATE SKIP LOCKED`) so multiple instances don't publish the same batch
- Mark events as published only after successful broker delivery
- Combine with the `DomainEventPublisher` pattern (`patterns:applying-domain-event-publisher`): the outbox write is one more handler, the application service stays unchanged
- Serialise only what consumers need — never secrets carried by the entity

**DON'T:**
- Publish to Kafka and write to DB in separate transactions (dual-write problem - one can succeed while the other fails)
- Delete outbox entries immediately after publishing - keep them for debugging/auditing and clean up with a scheduled job
- Process outbox entries without ordering guarantees - use `created_at` ordering per aggregate and the aggregate id as message key
- Skip idempotency in consumers - at-least-once delivery means duplicates will happen

## Testing
- Integration-test the outbox write against the real database and assert that a **rolled-back transaction leaves no outbox row** — the test that proves the pattern
- Test the poller: published entries are marked, a broker failure leaves them unpublished

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
