---
name: handling-errors-with-either
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing any code that uses an Either / Result type for error handling. This includes adding, modifying, or fixing Either return types (Arrow Either, discriminated-union Result), sealed / union error types, either {} builders and .bind() calls, early returns on Left, fold() mappings, or Left/Right handling in any part of the code.
---

## Purpose
Use an `Either<Error, Success>` type as an alternative to exceptions for domain error handling. Errors become explicit return values, making failure paths visible in function signatures and avoiding exception-based control flow.

## Strategy
1. Domain functions return `Either<DomainError, Result>` instead of throwing exceptions
2. Application services compose operations sequentially, short-circuiting on the first `Left`
3. Controllers unwrap Either results and map to HTTP responses
4. Infrastructure failures (DB, HTTP client) still use exceptions - Either is only for domain errors

## Guidelines

**DO:**
- Use closed (sealed / discriminated-union) types for errors (e.g., `CreateTeamError`)
- Compose multiple Either-returning operations in application services with a sequential, short-circuiting construct (a builder/comprehension, or an early return on `Left`) so the happy path reads top to bottom
- Keep error types specific to each domain function (e.g., `CreateTeamError`, `AddMemberError`), add enums/string literals inside for more granular information
- Map Either results to HTTP responses in controllers with an exhaustive match over the error variants
- Use `Left` for domain errors, `Right` for success

**DON'T:**
- **NEVER** return generic Domain errors from domain functions or application services, use specific ones
- Chain `.map`, `.flatMap`, `.onRight` - use the sequential construct instead for readability
- Return Either from infrastructure functions - use exception-based error handling instead
- Use Either for infrastructure failures - those should remain as exceptions, handled by the global exception handler
- Create a single global error type for all use cases - keep error types scoped

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
