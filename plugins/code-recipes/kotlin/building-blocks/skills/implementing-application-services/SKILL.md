---
name: implementing-application-services
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing services (use cases) classes in the application layer. 
---

## Purpose
Application services orchestrate infrastructure and domain to execute business use cases. They coordinate operations without containing business logic.

## Typical Flow
1. Receive input from infrastructure (controllers)
2. Retrieve or create entity from repository
3. Call external services if needed for additional data
4. Execute domain methods to perform business actions
5. Persist changes through repository
6. Publish domain events to signal what happened
7. Return results

## Guidelines

**DO:**
- Work on a single aggregate/entity per service (coordinating multiple aggregates belongs in callers or async handlers)
- Use constructor-based dependency injection
- Name with single action verb: `CreateTeamService`
- Use primitive types for parameters (create a command/request class in same file if more than 5 params)

**DON'T:**
- Depend on other application services (represents coupling between use cases; extract to domain instead)
- Include business rules (if/when/loops/checks of domain constraints) (belongs in domain layer)
- Add logging (use domain event handlers or infrastructure boundaries like controllers/repositories)
- Chain kotlin scope functions or library calls that obscure the orchestration flow (e.g. arrow's Either chains of map, flatMap onRight or let, also, run from kotlin)
- Allow nested private methods, just one level from the main function

## Spring specific guidelines:
- Keep service spring-agnostic as possible, but allow pragmatic use of Spring annotations when needed for transactions or other concerns:
  - Use `@Service` annotation to mark application services as Spring beans.
  - Use `@Transactional` annotation for transaction management when needed.

## Examples
Please use always these examples as reference: [examples.md](examples.md)
