# Infrastructure Layer - Outbound Adapters (`**/infra/outbound/*`)

Outbound adapters implement outbound ports defined in the domain layer. They translate domain calls to external systems (databases, APIs, message brokers).

## Core Principles

- **No business logic** - only translation, mapping, and external system integration
- **Implement domain ports** - Implement interfaces defined in domain layer
- Use **constructor injection**
- File naming: `<ExternalSystem><PortConcern>`, e.g., `PostgresqlTeamRepository`, `HttpClientPeopleFinder`, `KafkaTeamEventProducer`
- Use `@Repository` for database repositories, `@Component` for other adapters
- Declare DTOs in the same file where they are used for cohesion
- DTOs are pure data classes with no logic
- Use internal private functions for simple mapping, dedicated mapper classes for complex transformations
- Don't create subfolders unless files grow too much (>10), then group by external system (db, http, kafka, aws)

## Rules

**Exception Handling:**
- Don't throw domain exceptions - those belong in domain/application layers
- Return `null` instead of throwing when entity not found
- Map framework/library exceptions to infrastructure exceptions only when needed:
    - When signaling infrastructure errors (not business logic)
    - When they'll be caught/handled in other infrastructure layers
    - Examples: `OptimisticLockingException`, `HttpCallNonSucceededException`

**Configuration:**
- Constructor dependencies (JdbcTemplate, KafkaTemplate, Retrofit) should be auto-configured by Spring or defined in `infra/config`

---

## Database Repositories

Database repositories implement repository ports for data persistence.

### Rules

- Keep database schema simple - avoid complex joins, views, stored procedures
- Add always created_at and updated_at timestamps to all tables
- Push complexity to domain layer, not database
- Use optimistic locking when updating entities to prevent lost updates, with version column
- Transactions declared in application services layer, not here
- Use parameterized queries or prepared statements to prevent SQL injection
- Return `null` when entity not found (don't throw exceptions)
- Map database rows to domain entities using private extension functions

---

## HTTP API Clients

HTTP clients call external REST APIs and return domain objects.

### Rules

- Implement finder/retrieval ports for external services
- Return `null` for 404 responses (not found)
- Throw infrastructure exceptions for non-successful responses (except 404)
- Map external API DTOs to domain entities using private functions
- Define Retrofit/HTTP client interfaces in the same file or separate if reused

---

## Message Producers

Message producers publish events/messages to message brokers (Kafka, RabbitMQ).

### Rules

- Implement producer/publisher ports
- Serialize domain events/objects to message format
- Handle producer errors appropriately (log, retry, circuit breaker)
- Use templates (KafkaTemplate, RabbitTemplate) provided via constructor injection

---

## Domain Event Publishers

Domain event publishers dispatch domain events to subscribers for side effects (logging, metrics, external messaging).

### Rules

- **Always** implement `InMemoryDomainEventPublisher` pattern
- Dispatch events synchronously to all registered subscribers
- **Always** implement these subscribers:
    - `LoggingDomainEventSubscriber` - logs all domain events
    - `MetricsDomainEventSubscriber` - records metrics/counters for events
- **Always ask** if Kafka publishing is needed for domain events
- Keep implementations in same file if small, separate files if they grow (more than 400 lines)

---

## Examples

### Example: Database Repository

```kotlin
@Repository
class PostgresTeamRepository(private val jdbcTemplate: JdbcTemplate) : TeamRepository {

    override fun find(teamId: UUID): Team? = try {
        jdbcTemplate.queryForObject(
            "SELECT id, name, version FROM team WHERE id = ? FOR UPDATE",
            teamId
        ) { rs, _ -> rs.asTeam() }
    } catch (e: EmptyResultDataAccessException) {
        null
    }

    override fun save(team: Team) {
        val rows = jdbcTemplate.update(
            """
            INSERT INTO team (id, name, version) VALUES (?, ?, 0)
            ON CONFLICT (id) DO UPDATE SET name = EXCLUDED.name, version = team.version + 1
            WHERE team.version = ?
            """,
            team.teamId, team.teamName, team.version
        )
        if (rows == 0) throw OptimisticLockingException(team.teamId)
    }

    private fun ResultSet.asTeam() = Team(
        teamId = UUID.fromString(getString("id")),
        teamName = getString("name"),
        version = getInt("version")
    )
}

class OptimisticLockingException(teamId: UUID) : RuntimeException("Team $teamId was modified concurrently")
```

### Example: HTTP Client

```kotlin
@Component
class HttpClientPeopleFinder(private val peopleServiceApi: PeopleServiceApi) : PeopleFinder {

    @Throws(HttpCallNonSucceededException::class)
    override fun find(personId: UUID): Person? {
        val wrappedApiResponse = peopleServiceApi.find(personId.value).execute()
        val apiResponse = extractBody(wrappedApiResponse)
        return apiResponse?.let { Person(it.id, it.firstName, it.lastName) }
    }

    private fun extractBody(response: Response<PersonApiResponse>): PersonApiResponse? =
        when {
            response.isSuccessful -> response.body()!!
            response.code() == 404 -> null
            else -> throw HttpCallNonSucceededException(
                httpClient = this@PeopleServiceHttpClient::class.simpleName!!,
                errorBody = response.errorBody()?.charStream()?.readText()?.trimIndent(),
                httpStatus = response.code()
            )
        }
}

interface PeopleServiceApi {

    @GET("/people/{id}")
    fun find(@Path("id") accountId: UUID): Call<PersonApiResponse>
}

data class PersonApiResponse(val id: UUID, val firstName: String, val lastName: String)
```
### Example: Repository with Optimistic Locking

```kotlin
class PostgresTeamRepository(private val jdbcTemplate: JdbcTemplate) : TeamRepository {

    override fun find(teamId: UUID): Team? = try {
        jdbcTemplate.queryForObject(
            """ SELECT id, name, members FROM team WHERE id = ? FOR UPDATE """,
            teamId
        ) { rs, _ -> rs.asTeam() }
    } catch (exception: EmptyResultDataAccessException) {
        return null
    }

    override fun save(team: Team): Unit {
        val rows = jdbcTemplate.update(
            """
            INSERT INTO team (id, name, version) VALUES (?, ?, 0) ON CONFLICT (id) DO UPDATE
            SET name = EXCLUDED.name, version = team.version + 1 WHERE team.version = ?
            """,
            team.teamId.value,
            team.teamName.value,
            team.version
        )
        if (rows == 0) throw OptimisticLockException(team.teamId)
    }

    private fun ResultSet.asTeam() = Team(UUID.fromString(getString("id")), getString("name"))
}

class OptimisticLockException(teamId: UUID) : RuntimeException("Team $teamId was modified concurrently")
```

### Example: InMemoryDomainEventPublisher

```kotlin
@Component
class InMemoryDomainEventPublisher(
    private val subscribers: List<DomainEventSubscriber>
) : DomainEventPublisher {

    private val logger = LoggerFactory.getLogger(InMemoryDomainEventPublisher::class.java)

    override fun publish(event: DomainEvent) {
        logger.debug("Publishing domain event: {}", event::class.simpleName)
        subscribers.forEach { subscriber ->
            try {
                subscriber.handle(event)
            } catch (e: Exception) {
                logger.error("Subscriber ${subscriber::class.simpleName} failed to handle event ${event::class.simpleName}", e)
            }
        }
    }
}

interface DomainEventSubscriber {
    fun handle(event: DomainEvent)
}

@Component
class LoggingDomainEventSubscriber : DomainEventSubscriber {

    private val logger = LoggerFactory.getLogger(LoggingDomainEventSubscriber::class.java)

    override fun handle(event: DomainEvent) {
        logger.info("Domain event: {}", event)
    }
}

@Component
class MetricsDomainEventSubscriber(
    private val meterRegistry: MeterRegistry
) : DomainEventSubscriber {

    override fun handle(event: DomainEvent) {
        meterRegistry.counter("domain.events", "type", event::class.simpleName).increment()
    }
}
```