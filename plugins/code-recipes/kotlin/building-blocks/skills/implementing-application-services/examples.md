# Examples
For usage examples, see [examples.md](examples.md)

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
## Query Handler
```kotlin
@Service
class CanTeamAcceptMemberQueryHandler(
    private val teamRepository: TeamRepository
) {

    operator fun invoke(query: CanTeamAcceptMemberQuery): TeamCapacityView {
        val team = teamRepository.findById(query.teamId) ?: throw IllegalArgumentException("Team not found")
        return TeamCapacityView(team.id, team.canAcceptNewMember(), team.availableSpots())
    }
}
```

