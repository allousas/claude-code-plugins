# Kotlin Examples

## Aggregate root with an action verb
```kotlin
class Team private constructor(
    val id: UUID,
    val name: TeamName,
    val members: List<Member>,
    val version: Long
) {

    fun addMember(member: Member): Team {
        if (members.size >= MAX_MEMBERS) throw AddMemberException(MEMBER_LIMIT_REACHED, "Team $id has reached the maximum of $MAX_MEMBERS members")
        if (members.any { it.personId == member.personId }) throw AddMemberException(DUPLICATE_MEMBER, "Person ${member.personId} is already a member of team $id")
        return Team(id, name, members + member, version)
    }

    fun canAcceptNewMember(): Boolean = members.size < MAX_MEMBERS

    fun availableSpots(): Int = MAX_MEMBERS - members.size

    override fun equals(other: Any?): Boolean = other is Team && other.id == id

    override fun hashCode(): Int = id.hashCode()

    companion object {
        private const val MAX_MEMBERS = 10

        fun create(id: UUID, name: String): Team = Team(id, TeamName.create(name), emptyList(), 0)

        fun reconstitute(id: UUID, name: String, members: List<Member>, version: Long): Team =
            Team(id, TeamName.reconstitute(name), members, version)
    }
}

data class Member(val personId: UUID)
```

## Value object with validated creation
```kotlin
@JvmInline
value class TeamName private constructor(val value: String) {

    companion object {
        private const val MAX_LENGTH = 255

        fun create(value: String): TeamName {
            if (value.isBlank()) throw TeamValidationException(BLANK_NAME)
            if (value.length > MAX_LENGTH) throw TeamValidationException(TOO_LONG_NAME)
            return TeamName(value)
        }

        fun reconstitute(value: String): TeamName = TeamName(value)
    }
}
```

## Domain errors
```kotlin
data class AddMemberException(val type: AddMemberErrorType, override val message: String) : DomainException(message) {
    enum class AddMemberErrorType { MEMBER_LIMIT_REACHED, DUPLICATE_MEMBER }
}

data class TeamValidationException(val errorType: TeamValidationErrorType) : DomainException("Invalid team: $errorType") {
    enum class TeamValidationErrorType { BLANK_NAME, TOO_LONG_NAME }
}
```

## Domain events
```kotlin
sealed interface TeamEvent : DomainEvent { val teamId: UUID }

data class TeamCreated(val team: Team) : TeamEvent { override val teamId = team.id }

data class MemberAdded(override val teamId: UUID, val personId: UUID) : TeamEvent
```

## Application service acting on the new instance
```kotlin
@Service
class AddTeamMemberService(
    private val teamRepository: TeamRepository,
    private val domainEventPublisher: DomainEventPublisher
) {

    @Transactional
    open operator fun invoke(teamId: UUID, personId: UUID) {
        val team = teamRepository.find(teamId) ?: throw TeamNotFoundException(teamId)
        val updated = team.addMember(Member(personId))
        teamRepository.save(updated)
        domainEventPublisher.publish(MemberAdded(teamId, personId))
    }
}
```

## Kotlin specifics
- Make the primary constructor `private` and expose named factories in the `companion object` (`create` validates, `reconstitute` rehydrates) so the invalid path is unreachable
- Use `val` and read-only `List`/`Set` types; produce new instances instead of mutating (`members + member`). Avoid `data class` for entities with a private constructor - `copy` would bypass validation and equality must be by id
- Use `@JvmInline value class` for single-field value objects (no allocation overhead) and `data class` for multi-field ones (equality by value)
- Model lifecycle states and closed error/event sets with `enum class` or `sealed` hierarchies, so `when` is exhaustive
- No Spring annotations (`@Component`, `@Entity`, `@Transactional` ...) and no Jackson/JPA annotations in the domain package
