# Examples

## Spring Configuration class for http 
```kotlin
@Configuration
class HttpClientConfig {
    @Bean
    fun peopleServiceApi(
        defaultObjectMapper: ObjectMapper,
        circuitBreaker: CircuitBreaker,
        meterRegistry: MeterRegistry,
        @Value("\${clients.people-service.url}") url: String,
        @Value("\${clients.people-service.connectTimeoutMillis}") connectTimeout: Long,
        @Value("\${clients.people-service.readTimeoutMillis}") readTimeout: Long,
    ): PeopleServiceApi {
        val okHttpClient = OkHttpClient.Builder()
            .connectTimeout(connectTimeout, MILLISECONDS)
            .readTimeout(readTimeout, MILLISECONDS)
            .build()
        return Retrofit.Builder()
            .baseUrl(url)
            .addConverterFactory(JacksonConverterFactory.create(defaultObjectMapper))
            .addCallAdapterFactory(
                CircuitBreakerCallAdapter.of(
                    circuitBreaker
                ) { r -> !HttpStatus.valueOf(r.code()).is5xxServerError }
            )
            .client(okHttpClient)
            .build()
            .create(PeopleServiceApi::class.java)
    }

    // peopleFinder impl will just use @Component and constructor injection
}
```

## Constructor injection with defaults for testability
```kotlin
@Service
class CreateTeamService(
    private val teamRepository: TeamRepository,
    private val domainEventPublisher: DomainEventPublisher,
    private val generateId: () -> UUID = { UUID.randomUUID() } // default for production, override in tests
)
```

## Spring specifics
- `@Configuration` classes declare `@Bean` methods to create and wire infrastructure components
- `@ConfigurationProperties(prefix = "...")` binds YAML/properties to a data class
- Don't use `@Autowired` annotation. Use constructor-based injection instead
- Use spring starter modules to avoid manual configuration of common dependencies (e.g., spring-boot-starter-web, spring-boot-starter-data-jdbc, spring-boot-starter-kafka)
- Use `@Value` for simple configurations - use `@ConfigurationProperties` for complex configurations with multiple related properties
- Simple adapters just use `@Component`/`@Repository`/`@Service` with constructor injection; reserve `@Bean` methods for components that need programmatic construction
- Global cross-cutting concerns are picked up from the context once (e.g. a single `@ControllerAdvice`)
