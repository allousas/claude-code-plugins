# Kotlin Examples

## Spring Kafka Consumer
```kotlin
@Component
class TeamEventsKafkaConsumer(
    private val processTeamEvent: ProcessTeamEventService,
    private val objectMapper: ObjectMapper,
    private val logger: Logger = LoggerFactory.getLogger(TeamEventsKafkaConsumer::class.java)
) {
    @KafkaListener(topics = ["team-events"], groupId = "team-processor")
    fun consume(message: String) {
        try {
            val event = objectMapper.readValue(message, TeamEventMessage::class.java)
            if (event.eventType !in SUPPORTED_EVENT_TYPES) {
                logger.info("Ignoring unsupported event type: ${event.eventType}") // do NOT throw → no retry, no DLQ
                return
            }
            processTeamEvent(event.teamId, event.eventType)
            logger.info("Processed team event: ${event.eventId}")
        } catch (e: Exception) {
            logger.error("Failed to process team event", e)
            throw e // let Kafka error handler route to DLQ
        }
    }

    companion object {
        private val SUPPORTED_EVENT_TYPES = setOf("TEAM_CREATED", "TEAM_ARCHIVED")
    }
}

data class TeamEventMessage(val eventId: UUID, val teamId: UUID, val eventType: String)
```

## Spring kafka consumer replication data from another topic

```kotlin
@Component
class AccountReplicationKafkaConsumer(private val repository: AccountProjectionRepository, private val objectMapper: ObjectMapper) {

    @KafkaListener(topics = ["account-events"], groupId = "account-replication")
    fun consume(message: String) {
        val event = objectMapper.readValue(message, AccountEventMessage::class.java)
        repository.save(event.toProjection()) // access repository right away
    }
}
```

## Spring specifics
- Use `@KafkaListener` annotation
- Ensure `@EnableKafka` is enabled
- Use `KafkaConfig` class for separate configuration
- Ensure `DefaultErrorHandler` with exponential backoff is configured through `ConcurrentKafkaListenerContainerFactory`
- Ensure `DeadLetterPublishingRecoverer` is configured
