---
name: implementing-application-services
description: Apply when creating, modifying, or reviewing any class that is an application service, use case, command handler, or query handler in the services layer. 
---

## Purpose
Application services orchestrate infrastructure and domain to execute business use cases. They coordinate operations without containing business logic.

## Flow
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
- If a service is just a pass-through to an infrastructure component, e.g. call right away a repository method without changing the domain model, then it's not needed. Call infrastructure from infrastructure directly instead.

### Read services or query handlers

If the service is used to just read data, no business operation that performs a change, then it can be implemented as a query handler or read service. 
In that case:
- Use `QueryHandler` suffix instead of `Service` suffix or `UseCase` suffix
- Place the query handler in a `queries` package close at the same level of `services` if such package exists 
- Don't publish domain events

### Spring specifics:
- Keep service spring-agnostic as possible, but allow pragmatic use of Spring annotations when needed for transactions or other concerns:
  - Use `@Service` annotation to mark application services as Spring beans.
  - Use `@Transactional` annotation for transaction management when needed.


## Examples
Please use always these examples as reference: [examples.md](examples.md)
