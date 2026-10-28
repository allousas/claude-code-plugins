# Examples

## Fixtures

### Test builder
```kotlin
object TestBuilders {

    fun buildTeam(
        id: UUID = UUID.randomUUID(),
        name: String = "Teletubbies",
        version: Long = 0
    ) = Team(id, name, version)

    fun buildAnnualLeave(
        id: UUID = UUID.randomUUID(),
        employeeId: UUID = UUID.randomUUID(),
        year: Year = Year.of(2024),
        leaves: List<Leave> = emptyList(),
        version: Long = 0
    ) = AnnualLeave(id, employeeId, year, leaves, version)
}
```

### Postgres test container (one per test run)
```kotlin
object Postgres {

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

    val jdbcTemplate = JdbcTemplate(DriverManagerDataSource(container.jdbcUrl, container.username, container.password))

    fun truncate(vararg tables: String) = jdbcTemplate.execute("TRUNCATE ${tables.joinToString()}")
}

class KtPostgreSQLContainer : PostgreSQLContainer<KtPostgreSQLContainer>("postgres:latest")
```

## Domain Layer Tests

### Unit test with nested classes
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
        fun `should fail creating a team when name is too long`() {
            val teamName = "a".repeat(300)

            val exception = shouldThrow<TeamValidationException> { Team.create(UUID.randomUUID(), teamName) }

            exception.errorType shouldBe TOO_LONG_NAME
        }
    }
}
```

## Application Service Tests

### Unit test with mocked dependencies
```kotlin
class CreateTeamServiceTest {

    private val teamRepository = mockk<TeamRepository>()

    private val generateId = mockk<() -> UUID>()

    private val domainEventPublisher = mockk<DomainEventPublisher>(relaxUnitFun = true)

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

    @Test
    fun `should fail when repository rejects duplicate name`() {
        every { generateId() } returns UUID.randomUUID()
        every { teamRepository.save(any()) } returns TeamAlreadyExists.left()

        val result = createTeam(CreateTeamRequest("Existing Team"))

        result.shouldBeLeft(TeamAlreadyExists)
    }
}
```

## Infrastructure Tests

### Repository integration test
```kotlin
@Tag("integration")
class AnnualLeaveRepositoryTest {

    private val repository = AnnualLeaveRepository(Postgres.jdbcTemplate, defaultObjectMapper)

    @BeforeEach
    fun cleanUp() {
        Postgres.truncate("annual_leaves")
    }

    @Test
    fun `should save and find an annual leave`() {
        val annualLeave = TestBuilders.buildAnnualLeave()

        repository.save(annualLeave)
        val result = repository.findBy(annualLeave.employeeId, annualLeave.year)

        result shouldBe annualLeave.copy(version = 1)
    }

    @Test
    fun `should not find an annual leave when it does not exist`() {
        repository.findBy(UUID.randomUUID(), Year.of(2042)) shouldBe null
    }

    @Test
    fun `should throw OptimisticLockingException when version does not match`() {
        val annualLeave = TestBuilders.buildAnnualLeave()
            .also(repository::save)
            .let { repository.findBy(it.employeeId, it.year)!! }
        val staleVersion = annualLeave.copy(version = 3)

        shouldThrow<OptimisticLockingException> { repository.save(staleVersion) }
    }
}
```

### HTTP client integration test with WireMock
```kotlin
@Tag("integration")
@TestInstance(PER_CLASS)
class PublicHolidaysClientTest {

    private val wiremock = WireMockRule(wireMockConfig().dynamicPort()).also { it.start() }

    private val client = PublicHolidaysClient(
        httpClient = OkHttpClient(),
        objectMapper = defaultObjectMapper,
        baseUrl = "http://localhost:${wiremock.port()}"
    )

    @Test
    fun `should fetch public holidays`() {
        wiremock.stubFor(
            get(urlEqualTo("/api/v3/PublicHolidays/2024/DE"))
                .willReturn(okJson("""[{"date": "2024-12-25", "name": "Christmas Day"}]"""))
        )

        val result = client.fetch(Year.of(2024), "DE")

        result shouldBe listOf(MonthDay.of(12, 25))
    }

    @AfterAll
    fun tearDown() {
        wiremock.stop()
    }
}
```

### Controller integration test
```kotlin
@WebMvcTest(TeamsHttpController::class)
class TeamsHttpControllerTest(@Autowired private val mockMvc: MockMvc) {

    @MockkBean
    private lateinit var createTeamService: CreateTeamService

    @Test
    fun `should create a team`() {
        every { createTeamService(any()) } returns CreateTeamResponse(UUID.randomUUID()).right()

        mockMvc.perform(
            post("/teams").contentType(APPLICATION_JSON).content("""{"name": "Teletubbies"}""")
        ).andExpect(status().isCreated)
    }

    @Test
    fun `should return 400 when team name is invalid`() {
        every { createTeamService(any()) } throws TeamValidationException(TOO_LONG_NAME)

        mockMvc.perform(
            post("/teams").contentType(APPLICATION_JSON).content("""{"name": "${"a".repeat(300)}"}""")
        ).andExpect(status().isBadRequest)
    }
}
```

## Component Tests

### Base setup
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
        val wiremock = WireMockRule(wireMockConfig().dynamicPort()).also { it.start() }
    }

    class Initializer : ApplicationContextInitializer<ConfigurableApplicationContext> {

        override fun initialize(ctx: ConfigurableApplicationContext) {
            TestPropertyValues.of(
                "spring.datasource.url=" + Postgres.container.jdbcUrl,
                "spring.datasource.password=" + Postgres.container.password,
                "spring.datasource.username=" + Postgres.container.username,
                "spring.flyway.url=" + Postgres.container.jdbcUrl,
                "spring.flyway.password=" + Postgres.container.password,
                "spring.flyway.user=" + Postgres.container.username,
                "clients.internal.employees.baseUrl=http://localhost:${wiremock.port()}"
            ).applyTo(ctx.environment)
        }
    }
}
```

### Happy path E2E
```kotlin
class RequestLeaveComponentTest : BaseComponentTest() {

    @Test
    fun `should request a leave successfully`() {
        wiremock.stubEmployeeExists(employeeId = employeeId, countryCode = "DE")
        wiremock.stubPublicHolidays(year = 2024, countryCode = "DE")

        given()
            .port(servicePort)
            .contentType(APPLICATION_JSON)
            .body("""{"employeeId": "$employeeId", "days": ["12-20"], "year": 2024, "type": "VACATION"}""")
            .`when`()
            .post("/annual-leaves")
            .then()
            .statusCode(201)
    }
}
```

## Spring / Kotlin specifics
- Libraries: JUnit 5, kotest assertions (`shouldBe`, `shouldThrow`), mockk, Testcontainers, WireMock, RestAssured
- Name tests with backticks: `` fun `should create a team`() ``; group with `@Nested inner class`
- Relax mocks with `relaxUnitFun = true` in mockk when returning `Unit`
- JUnit creates a new test class instance per test by default, so mocks declared as fields are fresh per test
- Real implementations instead of mocks: `SimpleMeterRegistry` instead of mocking `MeterRegistry`; Testcontainers PostgreSQL instead of mocking `JdbcTemplate`; WireMock instead of mocking Retrofit/OkHttp; embedded Kafka or Testcontainers instead of mocking `KafkaTemplate`; `Clock.fixed()` instead of mocking `Clock`; `spyk` when a mock is unavoidable
- Share containers through a Kotlin `object` (started once per JVM, cleaned up by Ryuk) rather than per test class
- Use `@WebMvcTest` for controller integration tests with `@MockkBean` for service mocking — it includes `@ControllerAdvice` handlers, so error mapping is exercised through a real request
- Use `@SpringBootTest(webEnvironment = RANDOM_PORT)` for component tests with full context
- Use `@ActiveProfiles("test")` and `@DirtiesContext(classMode = AFTER_CLASS)` for component tests
- For stream consumers, schedulers, use `@SpringBootTest`, with narrowed context if possible
