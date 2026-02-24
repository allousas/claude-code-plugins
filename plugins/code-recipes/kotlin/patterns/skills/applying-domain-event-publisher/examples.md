# Examples

## Spring Kafka Producer
```kotlin
interface DomainEventPublisher {
    fun publish(event: DomainEvent)
}

interface DomainEventSubscriber {
    fun handle(event: DomainEvent)
}

class InMemoryDomainEventPublisher(val subscribers: List<DomainEventSubscriber>) : DomainEventPublisher {
    override fun publish(event: DomainEvent) = subscribers.forEach { it.handle(event) }
}

class LoggingSubscriber(
    private val logger: Logger = LoggerFactory.getLogger(MethodHandles.lookup().lookupClass())
) : DomainEventSubscriber {

    override fun handle(event: DomainEvent) {
        logger.info(event.logMessage())
    }

    private fun DomainEvent.logMessage(): String {
        val commonMessage = "domain-event: '${this::class.simpleName}', team-id: '${this.team.teamId.value}'"
        val specificMessage = when (this) {
            is TeamMemberJoined -> ", person-id: ${this.personId}"
            is TeamMemberLeft -> ", person-id: ${this.personId}"
            is TeamCreated -> ""
        }
        return "$commonMessage$specificMessage"
    }
}

class MetricsSubscriber(private val metrics: MeterRegistry) : DomainEventSubscriber {

    override fun handle(event: DomainEvent) {
        val tags = listOf(Tag.of("type", event::class.simpleName!!))
        metrics.counter("domain.event", tags).increment()
    }
}
```
