# Examples

## Sealed error type for a use case
```kotlin
sealed interface DomainError

data class CreateTeamError(val type: CreateTeamErrorType, val details: String? = null) : DomainError {
    enum class CreateTeamErrorType {
        INVALID_TEAM_NAME, // other error types could be added here
    }
}
```

## Domain function returning Either
```kotlin
class Team private constructor(val id: UUID, val name: String) {

    companion object {
        fun create(id: UUID, name: String): Either<DomainError, Team> = when {
                name.isBlank() -> CreateTeamError(type = INVALID_TEAM_NAME, details = "Name cannot be blank").left()
                else -> Team(id, name).right()
            }
    }
}
```

## Application service using either builder
```kotlin
@Service
class CreateTeamService(
    private val teamRepository: TeamRepository,
    private val domainEventPublisher: DomainEventPublisher,
    private val generateId: () -> UUID = { UUID.randomUUID() }
) {

    @Transactional
    open operator fun invoke(teamName: String): Either<CreateTeamError, UUID> = either {
        val newTeam = Team.create(generateId(), teamName).bind()
        teamRepository.save(newTeam)
        domainEventPublisher.publish(TeamCreated(newTeam))
        newTeam.id
    }
}
```

## Controller mapping Either to HTTP response
```kotlin
@RestController
@RequestMapping("/teams")
class TeamsHttpController(private val createTeam: CreateTeamService) {

    @PostMapping
    fun createTeam(@RequestBody request: CreateTeamRequest): ResponseEntity<*> =
        when (val result = createTeam(request.name)) {
            is Right -> 
                ResponseEntity.created(URI.create("/teams/${result.value}")).body(CreateTeamResponse(result.value))
            is Left -> when (result.value) {
                is CreateTeamError.INVALID_TEAM_NAME -> 
                    ResponseEntity.status(UNPROCESSABLE_ENTITY).body(ErrorResponse("INVALID_TEAM_NAME", result.value.reason))
                // other maps here
            }
        }
}
```
