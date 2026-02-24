# Application Service Test Examples

## Unit test with mocked dependencies
```kotlin
class CreateTeamServiceTest {

    private val teamRepository = mockk<TeamRepository>()

    private val generateId = mockk<() -> UUID>()

    private val domainEventPublisher = mockk<DomainEventPublisher>(relaxUnitFun = true)

    private val createTeam = CreateTeamService(teamRepository, domainEventPublisher, generateId)

    @Test
    fun `should create a team successfully`() {
        val team = buildTeam()
        every { generateId() } returns team.teamId.value
        every { teamRepository.save(team) } returns team.right()

        val result = createTeam(CreateTeamRequest(team.teamName.value))

        assertThat(result).isEqualTo(CreateTeamResponse(team.teamId.value).right())
        verify { domainEventPublisher.publish(TeamCreated(team)) }
    }

    @Test
    fun `should fail when repository rejects duplicate name`() {
        every { generateId() } returns UUID.randomUUID()
        every { teamRepository.save(any()) } returns TeamAlreadyExists.left()

        val result = createTeam(CreateTeamRequest("Existing Team"))

        result.shouldBeLeft(TeamAlreadyExists)
    }
}
```
