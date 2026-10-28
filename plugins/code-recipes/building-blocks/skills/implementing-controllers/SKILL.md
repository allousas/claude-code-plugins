---
name: implementing-controllers
description: ALWAYS invoke this skill when writing, changing, refactoring, planning (plan mode) or reviewing HTTP controllers or REST endpoints, or when the user asks about API or endpoint design. Covers routes, request/response DTOs, status codes and headers, and delegating to use cases or query handlers. Do not write controllers or endpoint code directly - use this skill first.
---

## Purpose
Adapters that translate external http requests into application service calls or repositories. They are thin translation layers with no business logic.

## Flow (HTTP Controllers)
1. Receive HTTP request with DTOs
2. Delegate to application service (commands) or query handler (queries)
3. Map result to response DTO
4. Return appropriate HTTP status code
5. Let exceptions propagate to the global exception handler

## Guidelines

**DO:**
- Keep adapters thin - delegate to application services or repositories, no business logic
- Declare DTOs in the same file where they are used for cohesion, below the controller; extract them to a separate file next to the controller only once shared beyond it
- DTOs are pure data structures with no logic
- Use small private functions for mapping/conversion in the same file; a separate mapper only if mapping is complex
- File naming: `<ExternalSystem><Concern>`, e.g., `TeamsHttpController`, following the project's file naming convention
- Model resources around domain entities (e.g., `/teams`, `/teams/{id}/members`), with nouns, pluralization and nesting
- Use proper HTTP methods (GET/POST/PUT/PATCH/DELETE) and return appropriate status codes: 201 for creation, 204 for no content etc ...
- Follow [RESTful principles](https://restfulapi.net/)
- Keep files small and focused

**DON'T:**
- Include business logic or orchestration - only translation, mapping, and error handling
- Use try-catch in controllers - delegate exception handling to the global exception handler
- Add validations at http dto level - delegate all validations to domain layer (transport-level parsing, e.g. a path param that must be a UUID, may be left to the framework)
- Leak HTTP specific dtos, HTTP concerns or framework related classes into the application services
- Do complex transformations

### Commands vs Queries in Controllers

**Commands (Write Operations):**
- Call command services / use cases from the application layer
- POST/PUT/PATCH/DELETE operations

**Queries (Read Operations) - two approaches:**
- **With orchestration/aggregation** - a projection, aggregation or custom shape is needed: call query handlers (see `building-blocks:implementing-query-handlers`)
- **Simple retrieval** - a domain object is returned as is: bypass application layer, call repositories/outbound ports directly

### Global Exception Handler (Required)

Every project MUST have a single global exception handler that:
1. Maps custom exceptions to HTTP status codes
2. Logs all exceptions with use case context extracted from stack trace
3. Returns consistent `ErrorResponse(code, message)` objects
4. Falls back to 500 with a generic message for unrecognized exceptions, without leaking internals

See `cross-cutting:handling-errors-with-exceptions` for the exception hierarchy.

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
