---
name: implementing-query-handlers
description: ALWAYS invoke this skill when writing, changing, refactoring, planning (plan mode) or reviewing query handlers, read services or read-only use cases, or when the user asks about data retrieval or separating reads from commands. Covers projections, views and aggregations. Do not write query handlers directly - use this skill first.
---

## Purpose
Query handlers retrieve and aggregate data without side effects. Read-only: no writes, no events.
Use them for read operations that need data shaping, projections or aggregations that do not fit directly in repositories or domain entities.
For simple reads that return domain entities without transformation, call repositories directly from http controllers, stream consumers or other adapters.

## Flow
1. Receive input from controller (or other inbound adapter)
2. Load data via repository / read model
3. Transform / project
4. Return result

## Guidelines

**DO:**
- Use `QueryHandler` suffix instead of `Service` or `UseCase` suffix, e.g. `CanTeamAcceptMemberQueryHandler`
- Place query handlers in a `queries` package/folder at the same level as the services/use-cases one, if such package exists
- Keep them stateless and function-like, with a single public entry point
- Use simple input types: primitives, or a query type (`CanTeamAcceptMemberQuery`) in the same file if needed
- Define the returned projection/view type in the same file, as an explicit, read-only type
- Keep transformation logic simple and focused on shaping data for the caller; ask the domain for derived facts (`team.canAcceptNewMember()`) instead of re-implementing rules
- Throw typed domain errors (e.g. `TeamNotFoundException`) when the queried entity does not exist

**DON'T:**
- Create pass-through query handlers - if a domain entity is returned without further transformation, call the repository directly from the adapter
- Write or change persisted state
- Publish domain events
- Depend on other services, use cases or query handlers
- Include direct infrastructure logic (HTTP, Kafka, SQL ...) - go through ports
- Use complex control flow or deep nesting

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
