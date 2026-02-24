# Examples

## Domain Exception hierarchy
```kotlin
sealed class DomainException(override val message: String) : RuntimeException(message)

data class TeamNotFoundException(val teamId:UUID) : DomainException("Team with id $teamId not found")

data class AddMemberException(val type: AddMemberErrorType, message: String) : DomainException(type, message) {
    enum class AddMemberErrorType { MEMBER_LIMIT_REACHED, DUPLICATE_MEMBER }
}
```

## Domain entity throwing exceptions
```kotlin
class Team(val id: UUID, val name: String, private val members: MutableList<Member>) {
    
    @Throws(AddMemberException::class)
    fun addMember(member: Member) {
        if (members.size >= MAX_MEMBERS) throw ValidationException(MEMBER_LIMIT_REACHED, "Team $id has reached the maximum of $MAX_MEMBERS members")
        if (members.any { it.personId == member.personId }) throw ValidationException(DUPLICATE_MEMBER, "Person ${member.personId} is already a member of team $id")
        members.add(member)
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
        team.addMember(Member(personId))
        teamRepository.save(team)
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
