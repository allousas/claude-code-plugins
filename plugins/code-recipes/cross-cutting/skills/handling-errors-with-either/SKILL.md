---
name: handling-errors-with-either
description: ALWAYS invoke this skill when writing, changing, refactoring, planning (plan mode) or reviewing code that returns or handles errors as Either / Result values, or when the user asks about modelling expected failures as values. Covers sealed or union error types, chaining and short-circuiting on Left, and mapping Left/Right at the boundary. Only when the project or operation uses Either/Result - otherwise use handling-errors-with-exceptions. Do not write Either-based error handling directly - use this skill first.
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
