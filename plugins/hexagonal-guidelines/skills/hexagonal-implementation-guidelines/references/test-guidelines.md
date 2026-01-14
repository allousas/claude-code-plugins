# Testing Guidelines

## Testing pyramid

- This project follows strictly the test pyramid, by which tests are categorized into three main types with different scopes and purposes:
  - **Unit tests** (mainly domain layer, but also complex mapping functions or edge case scenarios) - fast, isolated, check behaviour
  - **Integration tests** :
    - (infrastructure layer outbound adapters) - test integration with DB, external APIs, stream producers
    - (infrastructure layer inbound adapters) - test integration with web framework, request/response mapping
  - **Component tests**: blackbox end-to-end tests covering full flow from inbound to outbound adapters
```
        /\
       /  \        Component Tests (Isolated E2E)
      /----\       - Full flow testing
     /      \      - Happy paths only
    /--------\
   /          \    Integration Tests
  /            \   - Outbound adapters (DB, HTTP, Kafka)
 /--------------\  - Inbound adapters (Controllers)
/                \
/==================\ Unit Tests
                    - Domain layer (pure logic)
                    - Application services (mocked ports)
```

## General Testing Principles

- Write tests that are **clear, maintainable, and focused**
- Follow ** pattern**: Given, When, Then; without comments for them, just blank lines
- Avoid private methods/classes in tests
- Test behavior, not implementation details
- Keep tests **independent** - no shared state between tests
- Use **test doubles** (mocks, stubs) appropriately to isolate the unit under test
- Use **fixtures** for common setup code

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
- If there is the need to use an outbound port in the domain layer, implement your own test double for it
- Test domain rules, validations, and invariants

### Application Services Tests

- Test **use case orchestration**
- Mock outbound ports (repositories, external services)
- Verify correct interactions with domain and ports
- Test happy case and error scenarios
- Only test non-happy test scenarios that can be extracted from the signatures of the classes called (e.g., exceptions thrown by ports, domain exceptions, errors ...)

### Infrastructure Tests

#### Outbound Adapters

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

#### Inbound Adapters
- Test **API endpoints** using @WebMvcTest framework
- Test request/response mapping
- Test validation and error handling, only errors that are mapped at this layer
- Mock application services using @MockkBean
- For stream consumers, schedulers, use @SpringBootTest, with narrowed context if possible


## Component Tests
- Test **end-to-end flows** from inbound to outbound adapters
- Use real external systems with test containers or in-memory alternatives
- Test just happy paths and critical error scenarios, avoid exhaustive error testing here
- Test response codes, no payloads, and any relevant side effects (kafka messages ...)
- Don't check internal states or database states unless critical for the flow
- Use SpringBootTest with full context
- Name tests as `<FeatureBeingTested>ComponentTest`

## Test Coverage

- Aim for **90%+ code coverage**
- Focus on **critical paths** and **business logic**
- Don't chase 100% coverage - focus on value

## Best Practices

### ✅ DO:
- Test one behavior per test
- Use descriptive test names
- Keep tests simple and readable
- Use fixtures for common setup
- Test edge cases and error conditions
- Test domain logic thoroughly
- When mocking:
  - Relax mocks where possible, e.g., `relaxUnitFun = true` in mockk when returning Unit or relaxing functions when they are just stubs
  - Verify only side effects that are relevant to the behavior being tested (e.g., verify that a domain event was published or aggregate was saved, not every single interaction)
  
### ❌ DON'T:
- Test implementation details
- Create brittle tests that break on refactoring
- Use real external services in unit tests
- Share mutable state between tests
- Mock everything (especially domain objects)
- Write overly complex test setup

---

## Examples

### Domain Layer Test Example

```kotlin
class TeamTest {

    @Nested
    inner class CreateTeam {

        @Test
        fun `should create a team`() {
            val teamId = UUID.randomUUID()
            val teamName = "Teletubbies"

            val result = Team.create(teamId, teamName)

            result shouldBe Team(teamId, teamName)
        }

        @Test
        fun `should fail creating a team`() {
            val teamName = """
                Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut
                labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation ullamco laboris
                nisi ut aliquip ex ea commodo consequat.
                """

            val exception = shouldThrow<TeamValidationException> {
                Team.create(UUID.randomUUID(), teamName)
            }

            exception.errorType shouldBe TOO_LONG_NAME
        }
    }
    ...
}
```

### Application Service Test Example

```kotlin
class CreateTeamServiceTest {

    private val teamRepository = mockk<TeamRepository>()

    private val generateId = mockk<() -> UUID>()

    private val domainEventPublisher = mockk<DomainEventPublisher>(relaxedFun = true)

    private val createTeam = CreateTeamService(teamRepository, domainEventPublisher, generateId)

    @Test
    fun `should create a team successfully`() {
        val team = buildTeam()
        every { generateId() } returns team.teamId.value
        every { teamRepository.save(team) } returns team.right()

        val result = createTeam(CreateTeamRequest(team.teamName.value))

        assertThat(result).isEqualTo(CreateTeamResponse(team.teamId.value).right())
        verify { domainEventPublisher.publish(TeamCreated(team)) }
    }
    ...
}
```

### Infrastructure Test Examples

#### Outbound Adapter - Postgres Fixture

```kotlin
package mydomain.fixtures.containers

class Postgres() {

    val container: KtPostgreSQLContainer = KtPostgreSQLContainer()
        .withNetworkAliases("localhost")
        .withUsername("mydomain")
        .withPassword("mydomain")
        .withDatabaseName("mydomain")
        .also {
            System.setProperty("testcontainers.ryuk.container.timeout", "300")
            it.waitingFor(forListeningPort())
            it.start()
            Flyway(FluentConfiguration().dataSource(it.jdbcUrl, it.username, it.password)).migrate()
        }
    fun stop() {
        container.stop()
    }
}
class KtPostgreSQLContainer : PostgreSQLContainer<KtPostgreSQLContainer>("postgres:latest")
```

#### Outbound Adapter - Repository Test

```kotlin
@Tag("integration")
@TestInstance(PER_CLASS)
class AnnualLeaveRepositoryTest {

    private val postgres = Postgres()

    private val repository = AnnualLeaveRepository(postgres.jdbcTemplate, defaultObjectMapper)

    @Test
    fun `should save and find an annual leave`() {
        val annualLeave = TestBuilders.buildAnnualLeave()

        repository.save(annualLeave)
        val result = repository.findBy(annualLeave.employeeId, annualLeave.year)

        result shouldBe annualLeave.copy(version = 1)
    }

    @Test
    fun `should not find an annual leave when it does not exists`() {
        repository.findBy(UUID.randomUUID(), Year.of(2042)) shouldBe null
    }

    ...

    @Test
    fun `should throw OptimisticLockingException when updating a customer with wrong version`() {
        val annualLeave = TestBuilders.buildAnnualLeave()
            .also(repository::save)
            .let { repository.findBy(it.employeeId, it.year)!! }
        val updated = annualLeave.copy(version = 3)

        shouldThrow<OptimisticLockingException> { repository.save(updated) }
    }

    @AfterEach
    fun `tear down`() {
        postgres.stop()
    }
}
```

#### Component Test - Base Setup

```kotlin
@ActiveProfiles("test")
@SpringBootTest(webEnvironment = RANDOM_PORT)
@ContextConfiguration(initializers = [Initializer::class], classes = [App::class])
@DirtiesContext(classMode = AFTER_CLASS)
abstract class BaseComponentTest {

    init {
        RestAssured.defaultParser = Parser.JSON
    }

    @LocalServerPort
    protected val servicePort: Int = 0

    companion object {
        val postgres: Postgres = Postgres()
        val wiremock = WireMockRule(wireMockConfig().dynamicPort()).also { it.start() }
    }

    class Initializer : ApplicationContextInitializer<ConfigurableApplicationContext> {

        override fun initialize(configurableApplicationContext: ConfigurableApplicationContext) {
            TestPropertyValues.of(
                "spring.datasource.url=" + postgres.container.jdbcUrl,
                "spring.datasource.password=" + postgres.container.password,
                "spring.datasource.username=" + postgres.container.username,
                "spring.flyway.url=" + postgres.container.jdbcUrl,
                "spring.flyway.password=" + postgres.container.password,
                "spring.flyway.user=" + postgres.container.username,
                "clients.internal.employees.baseUrl=" + "http://localhost:${wiremock.port()}",
            ).applyTo(configurableApplicationContext.environment)
        }
    }
}
```