# Domain Layer (`**/domain`)

The domain layer is the **heart of the application**. It contains pure business logic, free from infrastructure concerns and framework dependencies.

## Core Concepts

### Aggregates & Entities

**Rules:**
- Enforce business rules within the aggregate
- Encapsulate behavior in methods (avoid anemic models)
- Immutable by default (return new instances instead of mutating)
- Keep models free of side effects
- Keep models simple, avoid parametric/generic types
- **Use composition over inheritance**
- Aggregate root controls all modifications to the aggregate

**When to use:**
- Core business concepts with identity (Team, Order, User)
- Concepts that change over time and need to enforce rules
- Objects that coordinate multiple related entities

### Value Objects

**Rules:**
- No identity, defined by values only
- Use with care, ask human if doubting
- Always immutable
- Contain validation logic and behavior related to the value
- Create only for concepts with specific rules/behaviors
- Don't overtype - use primitives if no validations needed
- **Avoid tiny types and unnecessary abstractions, prefer primitive types over over-typing**
- **CRITICAL: Value Objects MUST be constructed inside the aggregate that owns them** - Never construct value objects in application services
- **Throw domain exceptions for validation failures** - Use custom domain exceptions (e.g., `InvalidCodeException`) instead of generic `require()` or `IllegalArgumentException`

**When to use:**
- Concepts with clear validation rules defined in the functional requirements (Email, Money, DateRange)
- enumerations with behavior
- To avoid primitive obsession

**When NOT to use:**
- Simple primitives without validation (just use String, Int, etc.)
- When no behavior or rules are associated with the value
- If it comes from a trusted external system and has no domain-specific rules (we can trust its validity, for example an email coming from another service that already validated it)

### Domain Events

**Rules:**
- Declared in domain layer, instantiated/published in application services
- Past tense naming (TeamCreated, PaymentProcessed)
- Fat events - contain aggregate or relevant data, not just IDs
- Immutable

### Outbound Ports

**Rules:**
- Just interfaces, no implementations
- Named clearly by purpose (TeamRepository, PaymentGateway)
- No "Port" suffix
- Return domain objects or null, never infrastructure types

### Domain Exceptions

**Rules:**
- Extend a common base (DomainException)
- Named clearly (InsufficientFundsException, InvalidEmailException)
- Declared in domain layer
- Thrown from domain or application services, never from infrastructure

### Domain Services

**When to use:**
- Multi-entity operations
- Logic that doesn't belong to any specific aggregate
- Complex calculations involving multiple domain objects

**When NOT to use:**
- Use sparingly - most logic should be in aggregates
- Don't use as a catch-all for business logic

## Cross-Cutting Rules

**Purity & Dependencies:**
- Contains **pure business logic**, no framework dependencies
- No dependencies on other layers (application services, infrastructure)
- Keep all domain models free of side effects

**File Organization:**
- Domain events: Define in `DomainEvents.kt` (unless they grow too large)
- Outbound ports: Define in `OutboundPorts.kt` (unless they grow too large)
- Domain exceptions: Define in `DomainExceptions.kt` (unless they grow too large)
- Aggregates: Each aggregate root in its own file

## Examples

### Example: Aggregate Root (Team)

An aggregate that enforces business rules around team membership:

```kotlin
// Team.kt - Aggregate Root
data class Team(val teamId: UUID, val teamName: String, val members: Set<TeamMember>, val version: Int = 0) {
    companion object {
        private const val MAX_NAME_LENGTH = 100

        @Throws(TeamNameValidationException::class)
        fun create(teamId: UUID, teamName: String): Team {
            validateTeamName(teamName)
            return Team(teamId, teamName, emptySet())
        }

        private fun validateTeamName(name: String) {
            when {
                name.isBlank() -> throw TeamNameValidationException("Team name cannot be blank")
                name.length > MAX_NAME_LENGTH -> throw TeamNameValidationException("Team name too long")
            }
        }
    }

    @Throws(AlreadyPartOfTheTeamException::class)
    fun addMember(person: Person): Team {
        val existingMember = members.find { it.personId == person.personId }
        if (existingMember != null) throw AlreadyPartOfTheTeamException(person.personId)
        return this.copy(members = members + TeamMember(person.personId, person.name))
    }
    // other methods
}

// Person entity 
data class Person(val personId: UUID, val name: String, val email: String)
```

### Example: Value Object (NumericCode)

A value object that encapsulates validation logic with domain exceptions:

```kotlin
// NumericCode.kt - Value Object
data class NumericCode private constructor(val value: String) {

    companion object {
        private const val CODE_LENGTH = 4
        private val NUMERIC_REGEX = "^[0-9]+$".toRegex()

        @Throws(InvalidCodeException::class)
        fun create(code: String): NumericCode {
            when {
                code.length != CODE_LENGTH -> throw InvalidCodeException("Code must be exactly $CODE_LENGTH digits")
                !code.matches(NUMERIC_REGEX) -> throw InvalidCodeException("Code must contain only numbers")
            }
            return NumericCode(code)
        }
    }

    override fun toString(): String = value
}

// Domain exception for invalid code
class InvalidCodeException(message: String) : DomainException(message)
```

### Example: Value Object Construction Inside Aggregate

**CRITICAL:** Value objects must be constructed within the aggregate, not in application services:

```kotlin
// ✅ CORRECT: Value object created inside aggregate
data class Locker(val lockerId: UUID, val code: NumericCode?, val status: LockerStatus) {

    @Throws(InvalidCodeException::class)
    fun close(numericCode: String): Locker {
        val code = NumericCode.create(numericCode)  // VO created here, inside aggregate
        return this.copy(code = code, status = LockerStatus.CLOSED)
    }
}

// Application service receives primitives, passes to aggregate
@Service
class CloseLockerService(private val lockerRepository: LockerRepository) {

    @Transactional
    @Throws(InvalidCodeException::class)
    operator fun invoke(lockerId: UUID, code: String) {
        val locker = lockerRepository.find(lockerId) ?: throw LockerNotFoundException(lockerId)
        val closedLocker = locker.close(code)  // Aggregate creates the VO
        lockerRepository.save(closedLocker)
    }
}
```

```kotlin
// ❌ WRONG: Value object created in application service
@Service
class CloseLockerService(private val lockerRepository: LockerRepository) {

    @Transactional
    operator fun invoke(lockerId: UUID, code: String) {
        val numericCode = NumericCode.create(code)  // ❌ WRONG! VO created outside aggregate
        val locker = lockerRepository.find(lockerId) ?: throw LockerNotFoundException(lockerId)
        val closedLocker = locker.close(numericCode)
        lockerRepository.save(closedLocker)
    }
}
```

### Example: Domain Events

Events that signal important state changes:

```kotlin
// DomainEvents.kt
sealed interface DomainEvent

data class TeamCreated(val team: Team) : DomainEvent
```

### Example: Outbound Ports

Interfaces for interacting with external systems:

```kotlin
// Ports.kt
interface TeamRepository {
    fun find(teamId: UUID): Team?
    fun save(team: Team)
}

interface DomainEventPublisher {
    fun publish(event: DomainEvent)
}
```

### Example: Domain Exceptions

Exceptions that represent business rule violations:

```kotlin
// DomainExceptions.kt
sealed class DomainException(message: String) : RuntimeException(message)

class TeamNameValidationException(message: String) : DomainException(message)

class AlreadyPartOfTheTeamException(personId: UUID) : DomainException("Person $personId is already a member of the team")

class MemberNotFoundException(personId: UUID) : DomainException("Person $personId is not a member of the team")
```