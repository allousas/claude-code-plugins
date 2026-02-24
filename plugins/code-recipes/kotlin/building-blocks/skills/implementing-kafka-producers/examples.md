# Examples

## Spring Kafka producer for an integration message
```kotlin

@Component
class KafkaTeamEventProducer(
    private val kafkaTemplate: KafkaTemplate<String, ByteArray>,
    private val stream: String = "team.events"
) : TeamEventPublisher {

    override fun publish(event: TeamEvent) {
        kafkaTemplate.send(stream, event.teamId.toString(), message.toKafkaMessage())
    }
    
    private fun TeamCreatedEvent.toKafkaMessage() = ... // parse to specific message format
}
```