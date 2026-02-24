# Examples

## Entity with version field
```kotlin
class Team(val id: UUID, val name: String, val version: Int) {

    fun rename(newName: String): Team = Team(id, newName, version)

    companion object {
        fun create(id: UUID, name: String) = Team(id, name, version = 0)
    }
}
```

## OptimisticLockingException
```kotlin
class OptimisticLockingException(entityId: UUID) :
    RuntimeException("Concurrent modification detected for entity $entityId")
```

## Repository with version check
```kotlin
@Repository
class PostgresTeamRepository(private val jdbcTemplate: JdbcTemplate) : TeamRepository {
    
    override fun save(team: Team) {
        val rows = jdbcTemplate.update(
            """
            INSERT INTO team (id, name, version) VALUES (?, ?, 0)
            ON CONFLICT (id) DO UPDATE SET name = EXCLUDED.name, version = team.version + 1
            WHERE team.version = ?
            """,
            team.id, team.name, team.version
        )
        if (rows == 0) throw OptimisticLockingException(team.id)
    }
}
```
