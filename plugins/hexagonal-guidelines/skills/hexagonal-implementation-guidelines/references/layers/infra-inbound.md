# Infrastructure Layer - Inbound Adapters (`**/infra/inbound/*`)

Inbound adapters translate external signals/requests (HTTP, message queues, schedulers) into application service calls or direct outbound port calls.

## General Rules

- **Keep adapters thin** - delegate to application services
- **No business logic** - only translation, mapping, and error handling
- File naming: `<ExternalSystem><Concern>`, e.g., `TeamsHttpController`, `OrdersKafkaConsumer`, `ReportScheduler`
- Declare DTOs in the same file where they are used for cohesion
- DTOs are pure data classes with no logic
- Use internal private functions for mapping/conversion

---

## HTTP Controllers

HTTP controllers expose REST APIs for aggregates. They handle requests, delegate to application services, map responses, and handle exceptions.

### Rules

**Commands (Write Operations):**
- Call command services from `application/services`
- POST/PUT/PATCH/DELETE operations
- Modify system state

**Queries (Read Operations):**
- Two approaches:
  - **With orchestration/aggregation** → Call query handlers from `application/queries`
  - **Simple retrieval** → Bypass application layer, call repositories/outbound ports directly
- GET operations
- Read-only, no state changes

**When to bypass application layer for queries:**
- Single entity retrieval with no transformation, but create dto for response

**When to use query handlers:**
- Aggregating data from multiple repositories/services or complex data transformations or mappings

**General:**
- Model resources around domain aggregates (e.g., `/teams`, `/teams/{id}/members`)
- Don't use validations `@Valid` for format validation at DTO level
- Delegate all business validation to domain layer only
- **NEVER use try-catch in controllers** - delegate exception handling to `GlobalExceptionHandler`
- Return appropriate status codes: 201 for creation, 204 for no content
- See `error-handling.md` for global exception handler implementation

---

## Message Consumers

Message consumers listen to message queues (Kafka, RabbitMQ) and trigger application services.

**Rules:**
- Use `@KafkaListener` or `@RabbitListener` annotations
- Parse and validate incoming messages
- Call application services to trigger business use cases
- Could call repositories directly for data retrieval if needed
- Could call repositories directly for idempotency checks
- Could call repositories for data replication use cases
- Delegate to global configured mechanisms for retries and recovery
- Log processing for observability
- File naming: `<Topic><ExternalSystem>Consumer`, e.g., `OrdersKafkaConsumer`

---

## Schedulers

Schedulers trigger periodic tasks using `@Scheduled` annotation and call command services.

**Rules:**
- Use `@Scheduled` with cron expressions or fixed delays
- Can call repositories if needed for data retrieval
- Call command services to perform work
- Handle errors and log failures
- File naming: `<Concern>Scheduler`, e.g., `ReportGenerationScheduler`



## Examples

```kotlin
@RestController
@RequestMapping("/teams")
class TeamsHttpController(private val createTeam: CreateTeamService) {

    @PostMapping
    fun createTeam(@Valid @RequestBody request: CreateTeamRequest): ResponseEntity<CreateTeamResponse> {
        val teamId = createTeam(request.name)
        return ResponseEntity
            .created(URI.create("/teams/$teamId"))
            .body(CreateTeamResponse(teamId))
    }
}

data class CreateTeamRequest(val name: String)

data class CreateTeamResponse(val id: UUID)
```

```kotlin
@RestController
@RequestMapping("/teams")
class TeamsHttpController(private val addMember: AddTeamMemberService) {

    @PostMapping("/{teamId}/members")
    fun addMember(
        @PathVariable teamId: UUID,
        @Valid @RequestBody request: AddMemberRequest
    ): ResponseEntity<Unit> {
        addMember(teamId, request.personId)
        ResponseEntity.ok().build()
    }
}

data class AddMemberRequest(val personId: UUID)
```

### Example: Global Exception Handler (Required)

See `error-handling.md` for complete implementation with logging and use case extraction.

```kotlin
@ControllerAdvice
class GlobalExceptionHandler(
    private val logger: Logger = LoggerFactory.getLogger(GlobalExceptionHandler::class.java)
) {

    @ExceptionHandler(NotFoundException::class)
    fun handleNotFound(e: NotFoundException, request: HttpServletRequest): ResponseEntity<ErrorResponse> {
        logException(e, request)
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse(e.errorCode, e.message))
    }

    @ExceptionHandler(ConflictException::class)
    fun handleConflict(e: ConflictException, request: HttpServletRequest): ResponseEntity<ErrorResponse> {
        logException(e, request)
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(ErrorResponse(e.errorCode, e.message))
    }

    @ExceptionHandler(ValidationException::class)
    fun handleValidation(e: ValidationException, request: HttpServletRequest): ResponseEntity<ErrorResponse> {
        logException(e, request)
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY)
            .body(ErrorResponse(e.errorCode, e.message))
    }

    private fun logException(e: DomainException, request: HttpServletRequest) {
        val useCase = extractUseCaseFromStackTrace(e)
        logger.warn(
            "Domain exception | useCase={} | exception={} | message={} | path={} | method={}",
            useCase, e::class.simpleName, e.message, request.requestURI, request.method
        )
    }

    private fun extractUseCaseFromStackTrace(e: Throwable): String =
        e.stackTrace
            .firstOrNull { it.className.contains(".application.") && it.className.endsWith("Service") }
            ?.className?.substringAfterLast(".")
            ?: "UnknownUseCase"
}

data class ErrorResponse(val code: String, val message: String?)
```


### Example: Scheduler

```kotlin
@Component
class DailyReportScheduler(
    private val reportRepository: ReportRepository,
    private val generateDailyReport: GenerateDailyReportService,
    private val logger: Logger = LoggerFactory.getLogger(DailyReportScheduler::class.java)
) {

    @Scheduled(cron = "0 0 2 * * *") // Every day at 2 AM
    fun generateDailyReports() = try {
        logger.info("Starting daily report generation")
        val dailyReports = reportRepository.findReportsForToday()
        dailyReports.forEach { 
            logger.info("Report to generate: ${it.reportId}")
            generateDailyReport(it.id)
        }
        logger.info("Daily report generation completed")
    } catch (e: Exception) {
        logger.error("Failed to generate daily report", e)
    }
}
```

### Example: Kafka Consumer

```kotlin
@Component
class TeamEventsKafkaConsumer(
    private val processTeamEvent: ProcessTeamEventService,
    private val logger: Logger = LoggerFactory.getLogger(TeamEventsKafkaConsumer::class.java)
) {

    @KafkaListener(topics = ["team-events"], groupId = "team-processor")
    fun consume(message: String) = try {
        val event = parseEvent(message)
        processTeamEvent(event)
        logger.info("Processed team event: ${event.eventId}")
    } catch (e: Exception) {
        logger.error("Failed to process team event", e)
        throw e // let Kafka error handler route to DLQ
    }

    private fun parseEvent(message: String): TeamEventMessage = objectMapper.readValue(message, TeamEventMessage::class.java)
}

data class TeamEventMessage(val eventId: UUID, val teamId: UUID, val eventType: String)
```


