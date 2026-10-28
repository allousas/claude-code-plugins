# Examples

## Chained scope functions
```kotlin
// Don't
fun register(request: RegisterRequest) = request.let { validate(it) }.also { audit(it) }.run { repository.save(this) }

// Do
fun register(request: RegisterRequest): Team {
    val team = validate(request)
    audit(team)
    return repository.save(team)
}
```

## `when` instead of if-else chains
```kotlin
fun statusFor(error: AddMemberErrorType) = when (error) {
    MEMBER_LIMIT_REACHED -> UNPROCESSABLE_ENTITY
    DUPLICATE_MEMBER -> CONFLICT
}
```

## Named arguments for same-typed parameters
```kotlin
transfer(from = sourceAccountId, to = targetAccountId, amount = amount)
```

## Kotlin specifics
- Use `when` expressions for multi-branch logic; over sealed types/enums, omit `else` so the compiler enforces exhaustiveness
- Use `data class` for value objects and DTOs (`@JvmInline value class` for single-field wrappers)
- Use extension functions to add behavior to existing types without inheritance
- Prefer `val` over `var`, and read-only collections (`List`, `Map`) over mutable ones
- Don't chain scope functions (`let`, `also`, `run`, `apply`, `with`) - one level is fine
- Don't use `it` in multi-line lambdas - name the parameter explicitly
- Don't use nullable types (`T?`) when a value is always present; avoid `!!`
- Don't create utility classes/`object`s with only functions - use top-level or extension functions instead
