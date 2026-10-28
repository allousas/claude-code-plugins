---
name: applying-domain-event-publisher
description: Invoke when writing or reviewing code that publishes or reacts to domain events (including plans), or when the user asks about domain events. Covers the publisher port, in-memory publishers, in-process subscribers and their wiring. For reliable delivery to a broker use applying-outbox-pattern.
---
## What it is
A pattern that introduces an abstraction for publishing domain events, allowing the domain or application layer to emit events without depending on messaging infrastructure.

## Purpose
Publish domain events after state changes to notify other parts of the system about what happened, while keeping the domain layer decoupled from messaging infrastructure.

## Shape
- `DomainEventPublisher` — domain port the application service depends on
- In-memory dispatcher — implements the port and calls each handler in turn, never acts itself
- Handlers — one side-effect each (logging, metrics, in-process bus, outbox write)

## Typical Flow
1. Application service executes a domain operation (create, update, delete) and persists the state change
2. Application service calls `domainEventPublisher.publish` after the successful state change, inside the same transaction
3. Domain event is published to in-process handlers through an in-memory dispatcher
4. Handlers react to the event synchronously and **sequentially**, each performing one side-effect (outbox pattern, metrics, logging, etc.)
5. A handler failure fails `publish`, so the surrounding transaction rolls back and no side-effect is half-applied

## Guidelines

**DO:**
- Publish events in the application service after the state change succeeds, inside its transaction
- Define a `DomainEvent` type shared by all events; events are immutable data carriers with no behaviour, named in the past tense (`TeamCreated`, `MemberAdded`)
- Use an in-memory event dispatcher to pass the event to the different in-process handlers
- Use a separate handler per side-effect; co-locate the handlers in one file so the set of side-effects is visible at a glance
- Handle metrics, logging or publishing to a broker (via the transactional outbox) as handlers of this pattern
- Let a handler failure propagate and fail the whole `publish`
- If the stack passes transactions explicitly, keep the transaction handle on the `DomainEventPublisher` port even while no handler uses it — it is the seam the outbox slots into

**DON'T:**
- Use it for cross-service/process events - decouple asynchronous processing using messaging or queues (e.g. Kafka, SQS) through the outbox (`patterns:applying-outbox-pattern`)
- Publish events from inside domain entities - the application service decides when to publish
- Publish events before the state change is persisted (risk of publishing without actual change)
- Use events for request/response communication - events are fire-and-forget notifications
- Perform side-effects inside the dispatcher - it calls handlers, handlers act; collapsing both makes each untestable
- Add `handles` predicates, registries or dynamic routing until something actually varies at runtime - a handler that only cares about some events returns early
- Run handlers concurrently - a shared transaction handle is not safe for concurrent use
- Log or serialise the event wholesale - entities may carry secrets; log the event type and identifiers only

## Growing up
- **Metrics** — add a metrics handler once someone needs event rates (dashboards, alerting)
- **Outbox + message broker** — add an outbox handler when a consumer outside the service needs the event (`patterns:applying-outbox-pattern`)

## Testing
- Dispatcher: spy on the handlers; assert each receives the event, they run in order, and a failing handler propagates and stops later ones
- Handlers: unit-test each side-effect alone; assert no secret reaches logs or payloads

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
