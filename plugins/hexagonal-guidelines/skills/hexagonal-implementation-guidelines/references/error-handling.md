# Error Handling Guidelines

## Core Principle: Let It Crash

- **Catch only when you can handle** - If you can't handle the exception immediately, let it propagate up
- **Fail fast** - Let unexpected errors bubble up, don't suppress them, and crash early
- **Handle at the boundary** - Catch and handle exceptions at the uppermost layer (exception handlers, consumers)

## Exception Types

**Domain Exceptions:**
- Defined in `domain` layer
- Represent business rule violations (e.g., `TeamNotFoundException`, `AlreadyPartOfTheTeamException`)
- Thrown from domain or application services layer
- Inherit from common base class (e.g., `DomainException`)
- **Always** Use `@Throws` annotation for documentation and visibility
- Create specific domain exceptions for each validation rule (e.g., `InvalidCodeException` for code validation failures)

**Infrastructure Exceptions:**
- Defined in `infra` layer
- Represent infrastructure/technical errors (e.g., `OptimisticLockingException`, `HttpCallNonSucceededException`)
- Only throw when they'll be caught/handled in infrastructure layers
- Don't throw domain exceptions from infrastructure - return `null` instead

## Rules

**Where to catch:**
- Global exception handlers (`@ControllerAdvice`) - Centralized exception mapping for HTTP
- Message consumers - Log, metrics and rethrow
- Schedulers - Log, metrics and handle gracefully

**Where NOT to catch:**
- **Controllers** - Avoid use try-catch in controllers, delegate to global handler
- Domain layer - Let exceptions propagate
- Application services - Only catch to publish domain events for non-happy paths
- Repositories - Return `null` instead of throwing domain exceptions

**General:**
- Catch specific exceptions only, NEVER generic `Exception` or `Throwable`**
- Log exceptions with sufficient context for debugging
- Use `@Throws` annotation on public functions that throw custom exceptions, just for documentation and visibility
- **@Throws propagation:** When a public method calls another method annotated with `@Throws`, the caller MUST also be annotated with `@Throws` for the same exceptions. This ensures exception visibility throughout the call chain (e.g., if `Team.addMember()` throws `AlreadyPartOfTheTeamException`, then `AddTeamMemberService.invoke()` must also declare `@Throws(AlreadyPartOfTheTeamException::class)`)

## Anti-Patterns to Avoid

### ❌ NEVER: Generic try-catch in application services

```kotlin
// ❌ WRONG: Catching generic Exception
@Service
class CloseLockerService(private val lockerRepository: LockerRepository) {
    operator fun invoke(lockerId: UUID, code: String) {
        try {
            val locker = lockerRepository.find(lockerId)
            // ... business logic
        } catch (e: Exception) {  // ❌ NEVER catch generic Exception
            throw SomeOtherException("Failed to close locker")
        }
    }
}
```

### ✅ CORRECT: Let domain exceptions propagate, catch only specific ones when needed

```kotlin
// ✅ CORRECT: Let exceptions propagate, no try-catch needed
@Service
class CloseLockerService(private val lockerRepository: LockerRepository) {
    @Throws(LockerNotFoundException::class, InvalidCodeException::class)
    operator fun invoke(lockerId: UUID, code: String) {
        val locker = lockerRepository.find(lockerId) ?: throw LockerNotFoundException(lockerId)
        val closedLocker = locker.close(code)  // May throw InvalidCodeException - let it propagate
        lockerRepository.save(closedLocker)
    }
}
```

### ✅ CORRECT: Catch specific exception only to publish domain event

```kotlin
// ✅ CORRECT: Catch specific exception to publish event for non-happy path
@Service
class ProcessPaymentService(
    private val paymentRepository: PaymentRepository,
    private val domainEventPublisher: DomainEventPublisher
) {
    operator fun invoke(paymentId: UUID): ProcessPaymentResponse = try {
        // happy path
        val payment = paymentRepository.find(paymentId)!!
        val processed = payment.process()
        paymentRepository.save(processed)
        ProcessPaymentResponse.success(processed.id)
    } catch (e: InsufficientFundsException) {  // ✅ Specific exception
        domainEventPublisher.publish(PaymentFailed(paymentId, e.reason))
        ProcessPaymentResponse.failed(e.reason)
    }
}
```

---

## Global Exception Handler (Required for HTTP)

**Every project MUST have a single `GlobalExceptionHandler`** that:
1. Maps domain exceptions to HTTP status codes
2. Logs all exceptions with use case context extracted from stack trace
3. Returns consistent error responses

### Implementation

```kotlin
@ControllerAdvice
class GlobalExceptionHandler(
    private val errorMonitor: ErrorMonitor, 
    private val logger: Logger = LoggerFactory.getLogger(GlobalExceptionHandler::class.java)
) {

    @ExceptionHandler(TeamNotFoundException::class)
    fun handleNotFound(e: NotFoundException, request: HttpServletRequest): ResponseEntity<ErrorResponse> {
        errorMonitor.monitor(e, request)
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse(e.errorCode, e.message))
    }
    // other handlers...
}   

@Component
class ErrorMonitor(
    private val logger: Logger = LoggerFactory.getLogger(ErrorMonitor::class.java)
) {
    fun monitor(e: DomainException, request: HttpServletRequest?) {
        val useCase = extractUseCaseFromStackTrace(e)
        logger.warn(
            "Domain exception | useCase={} | exception={} | message={} | path={} | method={}",
            useCase,
            e::class.simpleName,
            e.message,
            request?.requestURI,
            request?.method
        )
    }

    private fun extractUseCaseFromStackTrace(e: Throwable): String =
        e.stackTrace
            .firstOrNull { it.className.contains(".application.") && it.className.endsWith("Service") }
            ?.className?.substringAfterLast(".")
            ?: "UnknownUseCase"
}

data class ErrorResponse(val code: String, val message: String?)
```

### Controller Pattern (Clean - No Try-Catch)

```kotlin
@RestController
@RequestMapping("/lockers")
class LockersHttpController(private val closeLockerService: CloseLockerService) {

    @PostMapping("/{lockerId}/close")
    fun closeLocker(
        @PathVariable lockerId: UUID,
        @RequestBody request: CloseLockerRequest
    ): ResponseEntity<Unit> {
        closeLockerService(lockerId, request.code)
        return ResponseEntity.noContent().build()
    }
    // Exceptions propagate to GlobalExceptionHandler automatically
}
```
