# Kotlin Examples

## Simple service with spring
```kotlin
@Service
class CreateTeamService(
    private val teamRepository: TeamRepository,
    private val domainEventPublisher: DomainEventPublisher,
    private val generateId: () -> UUID = { UUID.randomUUID() }
) {

    @Transactional
    open operator fun invoke(teamName: String): UUID {
        val newTeam = Team.create(generateId(), teamName)
        teamRepository.save(newTeam)
        domainEventPublisher.publish(TeamCreated(newTeam))
        return newTeam.id
    }
}
```

## Service using Either to handle errors
```kotlin
@Service
class CreateTeamService(
    private val teamRepository: TeamRepository,
    private val domainEventPublisher: DomainEventPublisher,
    private val generateId: () -> UUID = { UUID.randomUUID() }
) {

    @Transactional
    open operator fun invoke(teamName: String): Either<CreateTeamError, UUID> = either { // using either function instead of chains of map and flatMaps 
        val newTeam = Team.create(generateId(), teamName).bind()
        teamRepository.save(newTeam)
        domainEventPublisher.publish(TeamCreated(newTeam))
        newTeam.id
    }
}
```

## Kotlin specifics
- Expose the single entry point as `operator fun invoke(...)`, so callers read `createTeam(name)`
- Don't chain Kotlin scope functions (`let`, `also`, `run`, `apply`) or Arrow `Either` combinators (`map`, `flatMap`, `onRight`) to express the flow; use the `either { ... }` builder with `bind()` instead (see `cross-cutting:handling-errors-with-either`)

## Spring specifics
- Keep services as Spring-agnostic as possible, but allow pragmatic use of Spring annotations when needed for transactions or other concerns:
  - Use `@Service` annotation to mark application services as Spring beans
  - Use `@Transactional` annotation for transaction management (method must be `open`, or use the `kotlin-spring` plugin)
