---
name: implementing-application-services
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing application services, use cases or command handlers in the application layer (e.g. `CreateTeamService`, `CreateTeamUseCase`), including their transaction boundaries and event publishing.
---

## Purpose
Application services (use cases) orchestrate infrastructure and domain to execute a single business action. They coordinate operations without containing business logic.

## Flow
1. Receive input from infrastructure (controllers, consumers)
2. Retrieve or create entity from repository
3. Call external services if needed for additional data
4. Execute domain methods to perform business actions
5. Persist changes through repository
6. Publish domain events to signal what happened
7. Return results (if needed)

## Guidelines

**DO:**
- One service = one business action; name it with a single action verb plus the project's suffix: `CreateTeamService`, `CreateTeamUseCase`
- Work on a single aggregate/entity per service (coordinating multiple aggregates belongs in callers or async handlers)
- Keep services stateless and function-like, with a single public entry point
- Use constructor-based dependency injection; inject non-deterministic collaborators (id generators, clocks) so they can be replaced in tests
- Use primitive types for parameters (create a command/request type in the same file if more than 5 params)
- Return domain entities, ids or primitives - not DTOs, unless the domain does not have the needed shape; return nothing when the action has no result
- Own the transaction boundary: the whole action (load, change, persist, publish) runs in one transaction declared here
- Publish domain events through the domain event publisher port (see `patterns:applying-domain-event-publisher`)
- Keep orchestration explicit and linear

**DON'T:**
- Depend on other application services (represents coupling between use cases; extract to domain instead)
- Include business rules (if/when/loops/checks of domain constraints) (belongs in domain layer, see `building-blocks:implementing-rich-domain`)
- Include direct infrastructure logic (HTTP, Kafka, SQL ...) - go through ports
- Add logging or metrics (use domain event handlers or infrastructure boundaries like controllers/repositories)
- Chain functional combinators or helper calls that obscure the orchestration flow (e.g. chains of `map`/`flatMap` on result types)
- Allow nested private methods, just one level from the main function; no deep nesting or complex control flow
- Catch infrastructure exceptions to translate or wrap them - let them propagate to the boundary (see `cross-cutting:handling-errors-with-exceptions`)
- If a service is just a pass-through to an infrastructure component, e.g. call right away a repository method without changing the domain model, then it's not needed. Call infrastructure from infrastructure directly instead.

### Commands vs Queries
- Commands (write): modify state, persist changes, may publish events - this skill
- Queries (read): no state change, no events - see `building-blocks:implementing-query-handlers`

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
