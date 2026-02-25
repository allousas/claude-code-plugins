---
name: implementing-controllers
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing any file that is an HTTP controller, REST endpoint, API adapter, or global exception handler. This includes adding, modifying, or fixing request/response DTOs, status codes, route mappings, or error response handling.
---

## Purpose
Adapters that translate external http requests into application service calls or repositories. They are thin translation layers with no business logic.

## Flow (HTTP Controllers)
1. Receive HTTP request with DTOs
2. Delegate to application service (commands) or query handler (queries)
3. Map result to response DTO
4. Return appropriate HTTP status code
5. Let exceptions propagate to `GlobalExceptionHandler`

## Guidelines

**DO:**
- Keep adapters thin - delegate to application services or repositories, no business logic
- Declare DTOs in the same file where they are used for cohesion
- DTOs are pure data classes with no logic
- Use internal private functions for mapping/conversion
- File naming: `<ExternalSystem><Concern>`, e.g., `TeamsHttpController`
- Model resources around domain entities (e.g., `/teams`, `/teams/{id}/members`)
- Return appropriate status codes: 201 for creation, 204 for no content etc ...
- Follow [RESTful principles](https://restfulapi.net/)

**DON'T:**
- Include business logic - only translation, mapping, and error handling
- Use try-catch in controllers - delegate exception handling to `GlobalExceptionHandler`
- Add validations at http dto level - delegate all validations to domain layer
- Do not leak HTTP specific dtos, HTTP concerns or frameqrok related class into the application services

### Commands vs Queries in Controllers

**Commands (Write Operations):**
- Call command services from `application/services`
- POST/PUT/PATCH/DELETE operations

**Queries (Read Operations) - two approaches:**
- **With orchestration/aggregation** - Call query handlers or services if needed
- **Simple retrieval** - Bypass application layer, call repositories/outbound ports directly

### Global Exception Handler (Required)

Every project MUST have a single `GlobalExceptionHandler` that:
1. Maps custom exceptions to HTTP status codes
2. Logs all exceptions with use case context extracted from stack trace
3. Returns consistent `ErrorResponse(code, message)` objects

### Spring specifics
- Use `@ControllerAdvice` for global exception handling

## Examples
Please use always these examples as reference: [examples.md](examples.md)
