---
name: applying-domain-event-publisher
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing domain event publishing.
---
## What it is
A pattern that introduces an abstraction for publishing domain events, allowing the domain or application layer to emit events without depending on messaging infrastructure.

## Purpose
Publish domain events after state changes to notify other parts of the system about what happened, while keeping the domain layer decoupled from messaging infrastructure.

## Typical Flow
1. Application service executes a domain operation (create, update, delete)
2. Application service calls `domainEventPublisher.publish` after successful state change
3. Domain event is published to in-process handlers through an in-memory dispatcher
4. Event handlers/listeners react to the event synchronously (outbox pattern, metrics, logging, etc.)

## Guidelines

**DO:**
- Publish events in the application service after the state change succeeds
- Define a `DomainEvent` interface or base class for all events
- Use In-Memory event dispatcher for passing the event to the different in-process handlers
- Use a Separate Handler to handle the domain event 
- Handle metrics, logging or publishing to kafka (or tx-outbox) side-effects for domain events with this pattern and handlers


**DON'T:**
- Use For cross-service/process events - decouple asynchronous processing using messaging or queues (e.g. Kafka, SQS)
- Publish events from inside domain entities - the application service decides when to publish
- Publish events before the state change is persisted (risk of publishing without actual change)
- Use events for request/response communication - events are fire-and-forget notifications

## Examples
Please use always these examples as reference: [examples.md](examples.md)
