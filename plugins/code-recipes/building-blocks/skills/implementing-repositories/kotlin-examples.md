# Kotlin Examples

## Spring Database Repository
```kotlin
@Repository
class PostgresTeamRepository(private val jdbcTemplate: JdbcTemplate) : TeamRepository {

    override fun find(teamId: UUID): Team? = try {
        jdbcTemplate.queryForObject(
            "SELECT id, name, version FROM teams WHERE id = ? FOR UPDATE",
            teamId
        ) { rs, _ -> rs.asTeam() }
    } catch (e: EmptyResultDataAccessException) {
        null
    }

    override fun save(team: Team) {
        val rows = jdbcTemplate.update(
            """
            INSERT INTO teams (id, name, version, created_at, updated_at) VALUES (?, ?, 0, now(), now())
            ON CONFLICT (id) DO UPDATE SET name = EXCLUDED.name, version = team.version + 1, updated_at = now()
            WHERE team.version = ?
            """,
            team.id, team.name, team.version
        )
        if (rows == 0) throw OptimisticLockingException(team.id)
    }

    private fun ResultSet.asTeam() = Team(UUID.fromString(getString("id")), getString("name"), getInt("version"))
}
```

## Spring specifics
- Use `@Repository` annotation
- Use `JdbcTemplate` (or `NamedParameterJdbcTemplate`) for database access instead of JPA/Hibernate
- Translate `EmptyResultDataAccessException` into `null` for lookups that miss
- Repositories join the transaction opened by the service's `@Transactional`; never annotate repositories with it
