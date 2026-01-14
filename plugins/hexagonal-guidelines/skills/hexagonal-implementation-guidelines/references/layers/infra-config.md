# Infrastructure Layer - Configuration (`**/infra/config/*`)

Spring boot configuration, wiring, DI or complex configs that can not be done using spring annotations such as @Service, @Component or @Repository because they require extra config.

## Rules:
- Use `@Configuration` classes to define beans and configuration settings.
- Use `@Value` or `@ConfigurationProperties` to inject configuration values from application properties or environment variables.
- Keep configuration classes focused on wiring and configuration only.
- Avoid adding config when it can be done with simple annotations (@Service, @Component, @Repository) on classes.
- Use constructor injection for dependencies.
- Group related configuration settings together in the same class.
- Name configuration classes clearly to indicate their purpose (e.g. KafkaConfig, DatabaseConfig, RetrofitConfig).
- External http Apis clients configuration should be here (e.g., Retrofit), passing the retrofit apis already configured to the outbound adapters.

## Example:

```kotlin
@Configuration
class KafkaConfiguration {

    @Bean
    fun deadLetterPublishingRecoverer(bytesTemplate: KafkaTemplate<String, ByteArray>) =
        DeadLetterPublishingRecoverer(bytesTemplate)

    @Bean
    fun seekToCurrentErrorHandler(
        @Value("\${spring.kafka.error-handling.exponential-backoff.initial-interval}") initialInterval: Long,
        @Value("\${spring.kafka.error-handling.exponential-backoff.multiplier:1.5}") multiplier: Double,
        @Value("\${spring.kafka.error-handling.exponential-backoff.max-elapsed-time:30000}") maxElapsedTime: Long,
        deadLetterPublishingRecoverer: DeadLetterPublishingRecoverer,
    ): SeekToCurrentErrorHandler {
        val exponentialBackOff = ExponentialBackOff(initialInterval, multiplier)
            .apply { this.maxElapsedTime = maxElapsedTime }
        return SeekToCurrentErrorHandler(deadLetterPublishingRecoverer, exponentialBackOff)
    }
}
```