# Kotlin Examples

## Spring Kafka producer for an integration message
```kotlin
@Component
class KafkaTeamEventProducer(
    private val kafkaTemplate: KafkaTemplate<String, ByteArray>,
    private val objectMapper: ObjectMapper,
    private val stream: String = "team.events"
) : TeamEventPublisher {

    override fun publish(event: TeamEvent) {
        kafkaTemplate.send(stream, event.teamId.toString(), event.toKafkaMessage())
    }
    
    private fun TeamEvent.toKafkaMessage(): ByteArray = ... // map to message DTO and serialize with objectMapper
}
```

## Spring specifics
- Use `KafkaTemplate` to publish messages
- Delegate serialization to `ObjectMapper`
- Use `@Component` annotation to declare the bean
