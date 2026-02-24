# Test Fixtures Examples

## Test builder
```kotlin
object TestBuilders {

    fun buildTeam(
        id: UUID = UUID.randomUUID(),
        name: String = "Teletubbies",
        version: Long = 0
    ) = Team(id, name, version)

    fun buildAnnualLeave(
        id: UUID = UUID.randomUUID(),
        employeeId: UUID = UUID.randomUUID(),
        year: Year = Year.of(2024),
        leaves: List<Leave> = emptyList(),
        version: Long = 0
    ) = AnnualLeave(id, employeeId, year, leaves, version)
}
```
