# Kotlin Examples

All based on Spring Boot, if the project uses any other framework, use the equivalent classes/annotations.

## HTTP Controller (Command - Create)
```kotlin
@RestController
@RequestMapping("/teams")
class TeamsHttpController(private val createTeam: CreateTeamService) {

    @PostMapping
    fun createTeam(@RequestBody request: CreateTeamRequest): ResponseEntity<CreateTeamResponse> {
        val teamId = createTeam(request.name)
        return ResponseEntity
            .created(URI.create("/teams/$teamId"))
            .body(CreateTeamResponse(teamId))
    }
}

data class CreateTeamRequest(val name: String)

data class CreateTeamResponse(val id: UUID)
```

## HTTP Controller (Command - Operation on a resource)
```kotlin
@RestController
@RequestMapping("/teams")
class TeamsHttpController(private val addMember: AddTeamMemberService) {

    @PostMapping("/{teamId}/members")
    fun addMember(
        @PathVariable teamId: UUID,
        @RequestBody request: AddMemberRequest
    ): ResponseEntity<Unit> {
        addMember(teamId, request.personId)
        return ResponseEntity.noContent().build()
    }
}

data class AddMemberRequest(val personId: UUID)
```

## Global Exception Handler
Error codes and statuses live in the handler, never on the exception (see `cross-cutting:handling-errors-with-exceptions`):
```kotlin
@ControllerAdvice
class GlobalExceptionHandler(private val errorMonitor: ErrorMonitor) {

    @ExceptionHandler(TeamNotFoundException::class)
    fun handle(e: TeamNotFoundException, request: HttpServletRequest) =
        respond(e, request, HttpStatus.NOT_FOUND, "TEAM_NOT_FOUND")

    @ExceptionHandler(AddMemberException::class)
    fun handle(e: AddMemberException, request: HttpServletRequest) =
        respond(e, request, HttpStatus.UNPROCESSABLE_ENTITY, e.type.name)

    @ExceptionHandler(Exception::class)
    fun handle(e: Exception, request: HttpServletRequest) =
        respond(e, request, HttpStatus.INTERNAL_SERVER_ERROR, "INTERNAL_ERROR", "Internal server error")

    private fun respond(
        e: Exception,
        request: HttpServletRequest,
        status: HttpStatus,
        code: String,
        message: String? = e.message
    ): ResponseEntity<ErrorResponse> {
        errorMonitor.monitor(e, request)
        return ResponseEntity.status(status).body(ErrorResponse(code, message))
    }
}

@Component
class ErrorMonitor(private val meterRegistry: MeterRegistry) {
    private val logger = LoggerFactory.getLogger(ErrorMonitor::class.java)

    fun monitor(e: Exception, request: HttpServletRequest?) {
        val useCase = extractUseCaseFromStackTrace(e)
        logger.warn(
            "Exception | useCase={} | exception={} | message={} | path={} | method={}",
            useCase, e::class.simpleName, e.message, request?.requestURI, request?.method
        )
        meterRegistry.counter("errors", "type", e::class.simpleName!!, "useCase", useCase).increment()
    }

    private fun extractUseCaseFromStackTrace(e: Throwable): String =
        e.stackTrace
            .firstOrNull { it.className.contains(".application.") && it.className.endsWith("Service") }
            ?.className?.substringAfterLast(".")
            ?: "UnknownUseCase"
}

data class ErrorResponse(val code: String, val message: String?)
```

## Spring specifics
- Use `@RestController` + `@RequestMapping` for controllers and `ResponseEntity` to set status codes and headers
- Use `@ControllerAdvice` with `@ExceptionHandler` methods for global exception handling
- Keep request DTOs as plain `data class`es; don't use Bean Validation annotations (`@Valid`, `@NotBlank` ...) for business validations
