---
name: writing-tests
description: Invoke when writing or reviewing any test (including plans and tests added while implementing a feature), or when the user asks about test structure or conventions. Covers unit, integration and component tests, fixtures, fakes vs mocks, assertions, containerised dependencies and HTTP stubs.
---

## Purpose
Write clear, maintainable tests following the test pyramid. Each layer is tested at the altitude where its logic lives — business rules in the domain, orchestration in application services, transport in adapters — balancing confidence with speed.

## Test pyramid
- **Unit** (many, fastest): pure domain logic and application services in isolation, zero I/O — *is the logic right?*
- **Integration**: I/O boundaries and adapters against real engines (HTTP contracts, serialization, real DB queries) — *does the boundary speak the right protocol?*
- **Component** (few, slowest): the whole application as a runnable black box, end-to-end flows through its API and external side effects — *does the service do what was promised?*

## General Testing Principles

- Write tests that are **clear, maintainable, and focused**
- Follow the **Given, When, Then** pattern; without comments for them, just blank lines
- One behavior per test
- Avoid private methods/classes in tests
- Test behavior, not implementation details — assert observable outcomes (returned value, saved entity, published event, HTTP status) rather than internal calls when an outcome is available
- Keep tests **independent** - no shared mutable state between tests; reset mocks per test
- Use **test doubles** appropriately to isolate the unit under test:
    - Prefer **fakes** (small hand-written implementations of a port) when the test needs realistic behavior
    - Use mocks for stubbing or for verifying side effects
    - Don't mock the thing under test — it proves nothing
- Use **fixtures** for common setup code, kept in a shared test fixtures location and reused
- Prefer single assertions per test
- Assert the **specific** typed error, not a generic failure or a message string
- When mocking:
    - Relax mocks where possible (e.g. functions returning nothing, or functions that are just stubs)
    - Verify only side effects that are relevant to the behavior being tested (e.g., verify that a domain event was published or aggregate was saved, not every single interaction)
- Don't test framework internals or getters/setters
- Don't leave scratch notes or dead code in test files

## Test Structure

### Naming Conventions

- Test class/suite names: `<ClassBeingTested>Test` (e.g., `AddTeamMemberServiceTest`)
- Test names: `should <business behaviour>`, as a sentence (e.g. *should fail adding a member when the team is not found*), so a failure report reads as a list of broken promises
- Use descriptive names for tests to convey intent
- Group related tests using nested classes / nested suites when they grow large
- Use Test builders inside fixtures for complex object creation

## Testing by Layer

### Domain Layer Tests

- Test **pure business logic** in isolation — no mocks, no framework
- No mocking of domain entities or value objects
- Test domain rules, validations, and invariants: assert state transitions and that invariants throw the right typed error

### Application Services Tests

- Test **use case orchestration**
- Mock (or fake) outbound ports (repositories, external services)
- Verify side effects (e.g., domain events, database updates)
- Test happy case and error scenarios
- Only test non-happy test scenarios that can be extracted from the signatures of the classes called (e.g., exceptions thrown by ports, domain exceptions, errors ...)

### Infrastructure Tests

#### Outbound integrations

- Test **integration with external systems**
- Don't mock external systems - use test containers or in-memory alternatives
- **Never mock what you don't own** - when integrating with external libraries, systems or any package outside the project, use real implementations instead of mocks:
    - Metrics: use the library's in-memory registry instead of mocking it
    - Database: use a real engine in a container instead of mocking the DB client
    - HTTP clients: use a real stub HTTP server instead of mocking the HTTP client library
    - Kafka: use an embedded broker or a container instead of mocking the producer
    - Clock/Time: use a fixed clock instead of mocking it
    - For others: try to use real implementations or in-memory alternatives instead of mocking
    - If you really have to mock an external dependency, use spies instead of mocks to preserve real behavior as much as possible
- Test mapping between domain and external system DTOs
- For repositories, assert round-trip mapping and optimistic locking against the real engine
- Use fixtures for common setup (e.g., DB connections)
- Skip exhaustive error testing here, focus on happy paths and critical error scenarios
- Use programmatic setup/teardown for test containers in fixtures instead of relying on framework magic or annotations
- Let the test suite own its database: start it from the suite, apply migrations with the real migration tool, and publish its connection settings — a test run must carry no manual precondition (or fail fast with a usable instruction instead of a raw connection error)
- Start **one database per run**, shared across test classes, not one per test or per file — startup costs seconds
- Isolate tests by truncating tables before each test. Rolled-back-transaction isolation is faster but only works when the test controls the transaction — code that opens its own transactions or asserts real commit behaviour (optimistic locking) needs genuine commits
- Here mainly integration tests will be created, but if needed unit tests can be created for complex mapping functions

#### Inbound adapters (controllers, consumers)
- Test request/response mapping
- Test validation and error handling, only errors that are mapped at this layer
- Mock application services
- Exercise error-to-response mapping through a **real request** (with the global exception handler registered as in production), never by calling the handler directly — the mapping only exists at the transport boundary

## Component Tests
- Test **end-to-end flows** from inbound to outbound adapters, booting the full application
- Use real external systems with test containers or in-memory alternatives
- Test just happy paths and critical error scenarios, avoid exhaustive error testing here
- Test response codes, no payloads, and any relevant side effects (kafka messages ...)
- Don't check internal states or database states unless critical for the flow
- Name tests as `<FeatureBeingTested>ComponentTest`

## Test Coverage

- Aim for **90%+ code coverage**
- Don't chase 100% coverage - focus on value

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
