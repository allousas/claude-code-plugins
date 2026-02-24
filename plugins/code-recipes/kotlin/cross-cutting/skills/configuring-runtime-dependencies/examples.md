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
