---
name: handling-errors-with-either
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing any code that uses Arrow's Either type for error handling. This includes adding, modifying, or fixing Either return types, sealed error classes, either {} builders, .bind() calls, fold() mappings, or Left/Right handling in any part of the code.
---

## Purpose
Use Arrow's `Either<Error, Success>` as an alternative to exceptions for domain error handling. Errors become explicit return values, making failure paths visible in function signatures and avoiding exception-based control flow.

## Strategy
1. Domain functions return `Either<DomainError, Result>` instead of throwing exceptions
2. Application services use `either {}` builder with `.bind()` to compose operations
3. Controllers unwrap Either results and map to HTTP responses
4. Infrastructure failures (DB, HTTP client) still use exceptions - Either is only for domain errors

## Guidelines

**DO:**
- Use sealed classes/interfaces for error types (e.g., `sealed interface CreateTeamError`)
- Use Arrow's `either {}` builder in application services to compose multiple Either-returning operations
- Use `.bind()` inside `either {}` to short-circuit on errors (replaces flatMap chains)
- Keep error types specific to each domain function (e.g., `CreateTeamError`, `AddMemberError`), add enums/strings inside for more granular information
- Map Either results to HTTP responses in controllers using `fold()` or `when`
- Use `Either.Left` for domain errors, `Either.Right` for success

**DON'T:**
- **NEVER** return generic Domain errors from domain functions or application services, use specific ones
- Chain `.map`, `.flatMap`, `.onRight` - use `either {}` builder instead for readability
- Return Either from infrastructure functions - use exception-based error handling instead
- Use Either for infrastructure failures - those should remain as exceptions
- Create a single global error type for all use cases - keep error types scoped

### Spring specifics
- Controllers map `Either.Left` to appropriate HTTP error responses
- `GlobalExceptionHandler` still handles infrastructure exceptions

## Examples
Please use always these examples as reference: [examples.md](examples.md)
