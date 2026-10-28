# Examples

## Domain Exception hierarchy
```kotlin
sealed class DomainException(override val message: String) : RuntimeException(message)

data class TeamNotFoundException(val teamId: UUID) : DomainException("Team with id $teamId not found")

data class AddMemberException(val type: AddMemberErrorType, override val message: String) : DomainException(message) {
    enum class AddMemberErrorType { MEMBER_LIMIT_REACHED, DUPLICATE_MEMBER }
}
```

## Domain entity throwing exceptions
```kotlin
class Team(val id: UUID, val name: String, val members: List<Member>) {

    @Throws(AddMemberException::class)
    fun addMember(member: Member): Team {
        if (members.size >= MAX_MEMBERS) throw AddMemberException(MEMBER_LIMIT_REACHED, "Team $id has reached the maximum of $MAX_MEMBERS members")
        if (members.any { it.personId == member.personId }) throw AddMemberException(DUPLICATE_MEMBER, "Person ${member.personId} is already a member of team $id")
        return Team(id, name, members + member)
    }

    companion object {
        private const val MAX_MEMBERS = 10
    }
}
```

## Application service letting exceptions propagate
```kotlin
@Service
class AddTeamMemberService(
    private val teamRepository: TeamRepository,
    private val domainEventPublisher: DomainEventPublisher
) {

    @Transactional
    @Throws(AddMemberException::class, TeamNotFoundException::class)
    open operator fun invoke(teamId: UUID, personId: UUID) {
        val team = teamRepository.find(teamId) ?: throw TeamNotFoundException(teamId)
        val updated = team.addMember(Member(personId))
        teamRepository.save(updated)
        domainEventPublisher.publish(MemberAdded(teamId, personId))
    }
}
```

## Infrastructure exception (separate from domain)
```kotlin
class HttpCallNonSucceededException(
    val httpClient: String,
    val errorBody: String?,
    val httpStatus: Int
) : RuntimeException("HTTP call failed in $httpClient with status $httpStatus: $errorBody")
```

## Boundary handler: status, error code, log and metric
```kotlin
@ControllerAdvice
class GlobalExceptionHandler(private val meterRegistry: MeterRegistry) {

    private val logger = LoggerFactory.getLogger(javaClass)

    @ExceptionHandler(TeamNotFoundException::class)
    fun handle(exception: TeamNotFoundException, handler: HandlerMethod) =
        respond(exception, handler, NOT_FOUND, "TEAM_NOT_FOUND")

    @ExceptionHandler(AddMemberException::class)
    fun handle(exception: AddMemberException, handler: HandlerMethod) =
        respond(exception, handler, UNPROCESSABLE_ENTITY, exception.type.name)

    private fun respond(exception: Exception, handler: HandlerMethod, status: HttpStatus, errorCode: String): ResponseEntity<ErrorResponse> {
        val useCase = handler.method.name
        logger.warn("Use case '$useCase' failed with ${exception::class.simpleName}: ${exception.message}")
        meterRegistry.counter("errors", "type", exception::class.simpleName!!, "useCase", useCase).increment()
        return ResponseEntity.status(status).body(ErrorResponse(errorCode, exception.message))
    }
}
```

## Hot exception promoted to a per-operation sealed result
Only after monitoring and a benchmark show `AddMemberException` is hot:
```kotlin
sealed interface AddMemberResult {
    data class Added(val team: Team) : AddMemberResult
    data object MemberLimitReached : AddMemberResult
    data object DuplicateMember : AddMemberResult
}

fun addMember(member: Member): AddMemberResult = when {
    members.size >= MAX_MEMBERS -> AddMemberResult.MemberLimitReached
    members.any { it.personId == member.personId } -> AddMemberResult.DuplicateMember
    else -> AddMemberResult.Added(Team(id, name, members + member))
}
```

## Spring specifics
- Use `@ControllerAdvice` with `@ExceptionHandler` methods in `GlobalExceptionHandler`
- One handler method per exception type, mapping to the correct HTTP status and a stable API error code string
- Log all exceptions with use case context in the handler
- Inject `MeterRegistry` and increment an `errors` counter tagged by exception type and use case
- Mark methods that throw domain exceptions with `@Throws` — Kotlin has no checked exceptions, so this is purely for visibility
- Cost of exceptions on the JVM: `Throwable.fillInStackTrace()` walks the call stack at construction time. Benchmark the hot operation with JMH (throw-based vs. result-type variant) before promoting it to a sealed result
