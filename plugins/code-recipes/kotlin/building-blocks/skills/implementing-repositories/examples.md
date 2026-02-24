# Examples

## Spring Database Repository
```kotlin
@Repository
class PostgresTeamRepository(private val jdbcTemplate: JdbcTemplate) : TeamRepository {

    override fun find(teamId: UUID): Team? = try {
        jdbcTemplate.queryForObject(
            "SELECT id, name, version FROM team WHERE id = ? FOR UPDATE",
            teamId
        ) { rs, _ -> rs.asTeam() }
    } catch (e: EmptyResultDataAccessException) {
        null
    }

    override fun save(team: Team) {
        val rows = jdbcTemplate.update(
            """
            INSERT INTO team (id, name, version) VALUES (?, ?, 0)
            ON CONFLICT (id) DO UPDATE SET name = EXCLUDED.name, version = team.version + 1
            WHERE team.version = ?
            """,
            team.teamId, team.teamName, team.version
        )
        if (rows == 0) throw OptimisticLockingException(team.teamId)
    }

    private fun ResultSet.asTeam() = Team(UUID.fromString(getString("id")), getString("name"), getInt("version"))
}
```
