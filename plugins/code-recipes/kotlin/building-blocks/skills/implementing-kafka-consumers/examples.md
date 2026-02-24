# Examples

## Spring Kafka Consumer
```kotlin
@Component
class TeamEventsKafkaConsumer(
    private val processTeamEvent: ProcessTeamEventService,
    private val logger: Logger = LoggerFactory.getLogger(TeamEventsKafkaConsumer::class.java)
) {
    @KafkaListener(topics = ["team-events"], groupId = "team-processor")
    fun consume(message: String) = try {
        val event = objectMapper.readValue(message, TeamEventMessage::class.java)
        when (event.eventType) {
            "TEAM_CREATED" -> processTeamCreated(event)
            "TEAM_ARCHIVED" -> processTeamArchived(event)
            else -> {
                logger.info("Ignoring unsupported event type: ${event.eventType}") // do NOT throw → no retry, no DLQ
                return
            } 
        }
        logger.info("Processed team event: ${event.eventId}")
    } catch (e: Exception) {
        logger.error("Failed to process team event", e)
        throw e // let Kafka error handler route to DLQ
    }
}

data class TeamEventMessage(val eventId: UUID, val teamId: UUID, val eventType: String ...)
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