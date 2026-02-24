# Examples

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
        ResponseEntity.ok().build()
    }
}

data class AddMemberRequest(val personId: UUID)
```

## Global Exception Handler
```kotlin
@ControllerAdvice
class GlobalExceptionHandler(
    private val errorMonitor: ErrorMonitor,
    private val logger: Logger = LoggerFactory.getLogger(GlobalExceptionHandler::class.java)
) {

    @ExceptionHandler(NotFoundException::class)
    fun handleNotFound(e: NotFoundException, request: HttpServletRequest): ResponseEntity<ErrorResponse> {
        errorMonitor.monitor(e, request)
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(ErrorResponse(e.errorCode, e.message))
    }
    
    @ExceptionHandler(ValidationException::class)
    fun handleValidation(e: ValidationException, request: HttpServletRequest): ResponseEntity<ErrorResponse> {
        errorMonitor.monitor(e, request)
        return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(ErrorResponse(e.errorCode, e.message))
    }
}

@Component
class ErrorMonitor(private val logger: Logger = LoggerFactory.getLogger(ErrorMonitor::class.java)) {
    fun monitor(e: DomainException, request: HttpServletRequest?) {
        val useCase = extractUseCaseFromStackTrace(e)
        logger.warn(
            "Domain exception | useCase={} | exception={} | message={} | path={} | method={}",
            useCase, e::class.simpleName, e.message, request?.requestURI, request?.method
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
