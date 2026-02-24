# Infrastructure Test Examples

## Postgres test container fixture
```kotlin
class Postgres {

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

    fun stop() {
        container.stop()
    }
}

class KtPostgreSQLContainer : PostgreSQLContainer<KtPostgreSQLContainer>("postgres:latest")
```

## Repository integration test
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

    @AfterAll
    fun tearDown() {
        postgres.stop()
    }
}
```

## HTTP client integration test with WireMock
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

## Controller integration test
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
