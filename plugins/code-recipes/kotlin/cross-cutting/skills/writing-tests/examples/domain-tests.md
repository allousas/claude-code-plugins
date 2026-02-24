# Domain Layer Test Examples

## Unit test with nested classes
```kotlin
class TeamTest {

    @Nested
    inner class CreateTeam {

        @Test
        fun `should create a team`() {
            val teamId = UUID.randomUUID()
            val teamName = "Teletubbies"

            val result = Team.create(teamId, teamName)

            result shouldBe Team(teamId, teamName)
        }

        @Test
        fun `should fail creating a team when name is too long`() {
            val teamName = "a".repeat(300)

            val exception = shouldThrow<TeamValidationException> { Team.create(UUID.randomUUID(), teamName) }

            exception.errorType shouldBe TOO_LONG_NAME
        }
    }
}
```
