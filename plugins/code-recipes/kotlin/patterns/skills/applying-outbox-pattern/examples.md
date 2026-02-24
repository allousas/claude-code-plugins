# Examples

## Outbox table schema
```sql
CREATE TABLE outbox (
    id UUID PRIMARY KEY,
    aggregate_id UUID NOT NULL,
    event_type VARCHAR(255) NOT NULL,
    payload JSONB NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT now(),
    published BOOLEAN NOT NULL DEFAULT false
);

CREATE INDEX idx_outbox_unpublished ON outbox (created_at) WHERE published = false;
```

## OutboxRepository
```kotlin
@Repository
class PostgresOutboxRepository(private val jdbcTemplate: JdbcTemplate) {

    fun store(entry: OutboxEntry) {
        jdbcTemplate.update(
            "INSERT INTO outbox (id, aggregate_id, event_type, payload, created_at) VALUES (?, ?, ?, ?::jsonb, ?)",
            entry.id, entry.aggregateId, entry.eventType, entry.payload, entry.createdAt
        )
    }

    fun findUnpublished(limit: Int = 100): List<OutboxEntry> =
        jdbcTemplate.query(
            "SELECT id, aggregate_id, event_type, payload, created_at FROM outbox WHERE published = false ORDER BY created_at LIMIT ? FOR UPDATE SKIP LOCKED",
            limit
        ) { rs, _ -> rs.asOutboxEntry() }

    fun markAsPublished(ids: List<UUID>) {
        jdbcTemplate.update(
            "UPDATE outbox SET published = true WHERE id = ANY(?)",
            jdbcTemplate.dataSource!!.connection.createArrayOf("uuid", ids.toTypedArray())
        )
    }

    private fun ResultSet.asOutboxEntry() = OutboxEntry(
        id = UUID.fromString(getString("id")),
        aggregateId = UUID.fromString(getString("aggregate_id")),
        eventType = getString("event_type"),
        payload = getString("payload"),
        createdAt = getTimestamp("created_at").toInstant()
    )
}

data class OutboxEntry(val id: UUID, val aggregateId: UUID, val eventType: String, val payload: String, val createdAt: Instant)
```

## Outbox-based DomainEventPublisher adapter
```kotlin
@Component
class OutboxDomainEventPublisher(
    private val outboxRepository: PostgresOutboxRepository,
    private val objectMapper: ObjectMapper
) : DomainEventPublisher {

    override fun publish(event: DomainEvent) {
        val integrationEvent = event.toIntegrationEvent() // this could be protobuff, json, avro ...
        val entry = OutboxEntry(
            id = event.eventId,
            aggregateId = event.aggregateId(),
            eventType = event::class.simpleName!!,
            payload = objectMapper.writeValueAsString(integrationEvent),
            createdAt = event.occurredAt
        )
        outboxRepository.store(entry)
    }
} // if domain-event-publisher pattern with multiple in process handlers is used, this class could be a handler of DomainEvent
```

## Application service (unchanged - uses DomainEventPublisher port)
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
        domainEventPublisher.publish(TeamCreated(teamId = newTeam.id, teamName = teamName))
        return newTeam.id
    }
}
```

## Outbox poller publishing to Kafka
```kotlin
@Component
class OutboxPoller(
    private val outboxRepository: PostgresOutboxRepository,
    private val kafkaTemplate: KafkaTemplate<String, String>,
    private val topic: String = "domain-events",
    private val logger: Logger = LoggerFactory.getLogger(OutboxPoller::class.java)
) {

    @Scheduled(fixedDelay = 100)
    @Transactional
    open fun poll() {
        val entries = outboxRepository.findUnpublished()
        if (entries.isEmpty()) return

        entries.forEach { entry ->
            kafkaTemplate.send(topic, entry.aggregateId.toString(), entry.payload)
        }
        outboxRepository.markAsPublished(entries.map { it.id })
        logger.info("Published ${entries.size} outbox entries")
    }
}
```
