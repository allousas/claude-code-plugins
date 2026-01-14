# Application Services Layer

Implements use cases, orchestrating interactions between inbound adapters and the domain layer.

This layer is divided into two types of services:

## Command Services (Write Operations)

Located in `**/application/services` package.

**Command services** modify system state. They write data, change aggregates, enforce business rules, and publish domain events to signal state changes.

**Key characteristics:**
- Perform write operations (create, update, delete)
- Modify aggregate state through domain methods
- Persist changes to repositories
- Publish domain events to signal what happened
- May trigger side effects (notifications, auditing, etc.)
- Use transactions to ensure consistency

**When to use:**
- Business actions that enforce domain rules through aggregate changes

### Command Service Rules:
- **Avoid at all cost** service to service dependencies, this is a hard boundary, never do it.
- Prefer using function classes (kotlin classes with operator fun invoke)
- Class names should represent an action and end with "Service" for commands (e.g., CreateUserService) or "Query" for queries (e.g., GetTeamSummaryQuery)
- Import domain entities and ports
- Implement orchestration logic (no business rules)
- No ifs or loops that implement business rules (those belong in domain)
- May call multiple domain methods and outbound ports
- May handle transactions, using Spring's @Transactional annotation for transaction management (allowed pragmatic shortcut)
- Return `Unit` to represent pure side effects
- May raise domain exceptions
- When a domain event is needed for a non-happy case scenario, catch the exception, publish a domain event and return a meaningful response (Response dto from app service) object
- Catch domain exceptions only when you need to publish domain events for non-happy paths, otherwise let them propagate
- Publishes domain events through the domain event publisher port
- **Code around DDD aggregates**
- **Domain events are created and published here**, not in the domain layer

### Command Flow

- Receives input from inbound adapters (controllers)
- Creates or retrieves an aggregate root from a repository port
- Optionally calls external services through outbound ports if extra info is needed
- Calls domain methods on the aggregate root to perform business actions
- Persists changes through repository ports
- Creates and publishes domain events through the domain event publisher port
- Returns results to the inbound adapter
- Leak domain in and out to the infrastructure (e.g. Responses to controllers, calls to repositories ...)
- Create command dtos only when arguments grow too much (more than 5 parameters)
- Create response dtos only when a domain entity response is not fitting
- Command response dtos, if needed, should be declared in the same file as the application service

## Query Services (Read Operations)

Located in `**/application/queries` package.

**Query services** read system state without modifying it. They fetch data, aggregate information, and return it to callers.

**Key characteristics:**
- Perform read-only operations
- No state changes, no persistence, no domain events
- May aggregate data from multiple sources
- May transform domain entities into specialized DTOs
- No business logic, just data retrieval and transformation
- Can bypass this layer entirely for simple queries

**When to use:**
- Data aggregation from multiple repositories or services
- Complex data transformations or mappings
- Queries requiring orchestration logic

**When to skip this layer:**
- Simple single-entity retrieval
- Direct entity returns with no transformation
- No aggregation or orchestration needed
- Let inbound adapter call repository directly

### Query Handler Rules:
- Place in `/application/queries` package
- Class names should end with "Query" (e.g., GetTeamSummaryQuery)
- No domain events publishing
- No @Transactional unless read-only transaction needed for consistency
- May coordinate multiple repositories and outbound ports
- Focus on data aggregation and transformation, not business rules

### Query Flow

- Receives input from inbound adapters (controllers)
- Fetches data from one or more repositories or outbound ports
- Aggregates, transforms, or maps data into response DTOs
- Returns results to the inbound adapter
- No persistence, no domain events, no state changes
- Response DTOs should be declared in the same file as the query handler 

## Examples

### Example: Command Service (Create Operation)

Simple command that creates a new aggregate and publishes an event:

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
        domainEventPublisher.publish(TeamCreated(newTeam))
        return newTeam.id
    }
}
```

### Example: Command Service with Error Handling

When you need to publish domain events for non-happy paths:

```kotlin
@Service
class ProcessPaymentService(
    private val paymentRepository: PaymentRepository,
    private val domainEventPublisher: DomainEventPublisher
) {

    @Transactional
    open operator fun invoke(paymentId: UUID): ProcessPaymentResponse = try {
        val payment = paymentRepository.find(paymentId) ?: throw PaymentNotFoundException(paymentId)
        val paymentProcessed = payment.process()
        paymentRepository.save(paymentProcessed)
        domainEventPublisher.publish(PaymentProcessed(paymentProcessed))
        ProcessPaymentResponse.success(paymentProcessed.id)

    } catch (e: InsufficientFundsException) {
        domainEventPublisher.publish(PaymentFailed(paymentId, e.reason))
        ProcessPaymentResponse.failed(e.reason)
    }
}

data class ProcessPaymentResponse(val status: Status, val paymentId: UUID, val failureReason: String?) {
    enum class Status { SUCCESS, FAILED }

    companion object {
        fun success(paymentId: UUID) = ProcessPaymentResponse(Status.SUCCESS, paymentId, null)
        fun failed(reason: String) = ProcessPaymentResponse(Status.FAILED, null, reason)
    }
}
```

### Example: Query Service (With Aggregation)

When queries require orchestration, aggregation, or mapping logic, use application/queries:

```kotlin
// Located in application/queries package
@Service
class GetTeamSummaryQuery(private val teamRepository: TeamRepository, private val memberRepository: MemberRepository ) {

    operator fun invoke(teamId: UUID): TeamSummary {
        val team = teamRepository.find(teamId) ?: throw TeamNotFoundException(teamId)
        val members = memberRepository.findAllByTeamId(teamId)
        return TeamSummary.create(team, members)
    }
}
```
