---
name: writing-tests
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing tests in any layer.
---

## Purpose
Write clear, maintainable tests following the test pyramid. Each layer has a specific testing strategy that balances confidence with speed.
## General Testing Principles

- Write tests that are **clear, maintainable, and focused**
- Follow ** pattern**: Given, When, Then; without comments for them, just blank lines
- Avoid private methods/classes in tests
- Test behavior, not implementation details
- Keep tests **independent** - no shared state between tests
- Use **test doubles** (mocks, stubs) appropriately to isolate the unit under test
- Use **fixtures** for common setup code
- Prefer single assertions per test
- When mocking:
    - Relax mocks where possible, e.g., `relaxUnitFun = true` in mockk when returning Unit or relaxing functions when they are just stubs
    - Verify only side effects that are relevant to the behavior being tested (e.g., verify that a domain event was published or aggregate was saved, not every single interaction)


## Test Structure

### Naming Conventions

- Test class names: `<ClassBeingTested>Test` (e.g., `AssignLockerServiceTest`)
- Test method names: `should <business behaviour>`
- Use descriptive names for test methods to convey intent
- Group related tests using nested classes or regions if supported when they grow large
- Use Test builders inside fixtures for complex object creation

## Testing by Layer

### Domain Layer Tests

- Test **pure business logic** in isolation
- No mocking of domain entities or value objects
- Test domain rules, validations, and invariants

### Application Services Tests

- Test **use case orchestration**
- Mock outbound ports (repositories, external services)
- Verify side effects (e.g., domain events, database updates)
- Test happy case and error scenarios
- Only test non-happy test scenarios that can be extracted from the signatures of the classes called (e.g., exceptions thrown by ports, domain exceptions, errors ...)

### Infrastructure Tests

#### Outbound integrations

- Test **integration with external systems**
- Don't mock external systems - use test containers or in-memory alternatives
- **Never mock what you don't own** - when integrating with external libraries, systems or any package outside the project, try to use real implementations instead of mocks, examples:
    - Micrometer metrics: Use `SimpleMeterRegistry` instead of mocking `MeterRegistry`
    - Database: Use TestContainers with real PostgreSQL instead of mocking `JdbcTemplate`
    - HTTP clients: Use WireMock instead of mocking Retrofit/OkHttp
    - Kafka: Use embedded Kafka or TestContainers instead of mocking `KafkaTemplate`
    - Clock/Time: Use `Clock.fixed()` instead of mocking `Clock`
    - For others: Try to use real implementations or in-memory alternatives instead of mocking
    - If you really have to mock an external dependency, use spys instead of mocks to preserve real behavior as much as possible
- Test mapping between domain and external system DTOs
- Use **test containers** for databases
- Use Wiremock real to integrate and test http APIs in integration tests
- Use fixtures for common setup (e.g., DB connections)
- Skip exhaustive error testing here, focus on happy paths and critical error scenarios
- Use programmatic setup/teardown for test containers in fixtures instead of relying on spring or annotations
- Here mainly integration tests will be created, but if needed unit tests can be created for complex mapping functions

#### Inbound Integrations (Spring specifics)
- Test request/response mapping
- Test validation and error handling, only errors that are mapped at this layer
- Mock application services

## Component Tests
- Test **end-to-end flows** from inbound to outbound adapters
- Use real external systems with test containers or in-memory alternatives
- Test just happy paths and critical error scenarios, avoid exhaustive error testing here
- Test response codes, no payloads, and any relevant side effects (kafka messages ...)
- Don't check internal states or database states unless critical for the flow
- Use SpringBootTest with full context
- Name tests as `<FeatureBeingTested>ComponentTest`

### Spring specifics
- Use `@WebMvcTest` for controller integration tests with `@MockkBean` for service mocking
- Use `@SpringBootTest(webEnvironment = RANDOM_PORT)` for component tests with full context
- Use `@ActiveProfiles("test")` and `@DirtiesContext(classMode = AFTER_CLASS)` for component tests
- For stream consumers, schedulers, use @SpringBootTest, with narrowed context if possible

## Test Coverage

- Aim for **90%+ code coverage**
- Don't chase 100% coverage - focus on value

## Examples
Please use always these examples as reference:
- [Domain layer tests](examples/domain-tests.md)
- [Application service tests](examples/application-service-tests.md)
- [Infrastructure tests](examples/infrastructure-tests.md)
- [Component tests](examples/component-tests.md)
- [Test fixtures](examples/fixtures.md)