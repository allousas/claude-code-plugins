# Component Test Examples

## Base setup
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
        val postgres = Postgres()
        val wiremock = WireMockRule(wireMockConfig().dynamicPort()).also { it.start() }
    }

    class Initializer : ApplicationContextInitializer<ConfigurableApplicationContext> {

        override fun initialize(ctx: ConfigurableApplicationContext) {
            TestPropertyValues.of(
                "spring.datasource.url=" + postgres.container.jdbcUrl,
                "spring.datasource.password=" + postgres.container.password,
                "spring.datasource.username=" + postgres.container.username,
                "spring.flyway.url=" + postgres.container.jdbcUrl,
                "spring.flyway.password=" + postgres.container.password,
                "spring.flyway.user=" + postgres.container.username,
                "clients.internal.employees.baseUrl=http://localhost:${wiremock.port()}"
            ).applyTo(ctx.environment)
        }
    }
}
```

## Happy path E2E
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
