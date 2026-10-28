# Kotlin Examples

## Query Handler
```kotlin
@Service
class CanTeamAcceptMemberQueryHandler(
    private val teamRepository: TeamRepository
) {

    operator fun invoke(query: CanTeamAcceptMemberQuery): TeamCapacityView {
        val team = teamRepository.find(query.teamId) ?: throw TeamNotFoundException(query.teamId)
        return TeamCapacityView(team.id, team.canAcceptNewMember(), team.availableSpots())
    }
}

data class CanTeamAcceptMemberQuery(val teamId: UUID)

data class TeamCapacityView(val teamId: UUID, val canAcceptNewMember: Boolean, val availableSpots: Int)
```

## Wiring to a controller
```kotlin
@GetMapping("/{teamId}/capacity")
fun capacity(@PathVariable teamId: UUID): TeamCapacityView = canTeamAcceptMember(CanTeamAcceptMemberQuery(teamId))
```

## Kotlin specifics
- Expose the entry point as `operator fun invoke(...)`; return immutable `data class` views with `val` fields and read-only `List` collections

## Spring specifics
- Use `@Service` annotation; add `@Transactional(readOnly = true)` only if several reads must see a consistent snapshot
