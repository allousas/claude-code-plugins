---
name: handling-errors-with-exceptions
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing error handling using exceptions in any layer.
---

## Purpose
Define a consistent exception hierarchy so domain and infrastructure errors propagate cleanly to infrastructure boundaries where they are translated into appropriate responses (HTTP status codes, Kafka DLQ, etc.).

## Strategy
1. Domain layer throws domain-specific exceptions to signal business rule violations
2. Application services let domain exceptions propagate (no catch)
3. Infrastructure adapters (controllers, consumers) let exceptions reach global handlers
4. Infrastructure exceptions (DB failures, HTTP client errors) are separate from domain exceptions and defined close to the infrastructure layer

## Guidelines

**DO:**
- Apply the Let It Fail principle: do not defensively catch domain exceptions just to log or rethrow — allow them to propagate to the appropriate boundary (e.g., API layer or global handler)
- Create a sealed `DomainException` base class for all domain errors
- Mark any method throwing a domain exception as `throws`, just for minimal visibility
- Create specific exception subclasses for each business error case (e.g., `TeamCreationException`)
- Throw domain exceptions in domain entities or domain services only
- Let exceptions propagate naturally through application services without catching
- Use separate infrastructure exceptions for infra failures (DB, HTTP client, Kafka) (e.g.,`HttpFailureException`)
- Create infrastructure-specific exceptions only when you’re adding meaningful context or converting low-level failures into something your application understands; otherwise, let them propagate instead of wrapping them unnecessarily.
- Include meaningful error codes for API consumers (e.g., `"TEAM_NOT_FOUND"`, `"MEMBER_LIMIT_REACHED"`)

**DON'T:**
- Catch and rethrow domain exceptions in application services - let them propagate
- Use generic exceptions (`RuntimeException`, `IllegalStateException`) for domain errors
- Throw domain exceptions from infrastructure adapters (repositories, HTTP clients) - use infrastructure exceptions instead
- Use try-catch in controllers - delegate to `GlobalExceptionHandler`
- Mix domain and infrastructure exception hierarchies
- Don't catch exceptions just to log and rethrow - let them propagate to global handlers for consistent logging and response mapping

### Spring specifics
- Use `@ControllerAdvice` with `@ExceptionHandler` methods in `GlobalExceptionHandler`
- One handler method per exception type, mapping to the correct HTTP status
- Log all exceptions with use case context in the handler

## Examples
Please use always these examples as reference: [examples.md](examples.md)
