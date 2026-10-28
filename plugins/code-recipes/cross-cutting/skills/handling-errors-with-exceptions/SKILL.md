---
name: handling-errors-with-exceptions
description: ALWAYS invoke this skill when writing, changing, refactoring, planning (plan mode) or reviewing code that throws, catches or defines errors, or when the user asks how to model, propagate or map failures. Covers domain and infrastructure exception hierarchies, propagation, the global exception handler, error-to-response and error-code mapping, and promoting hot exceptions to result types. The default unless the project or operation uses Either/Result. Do not write error-handling code directly - use this skill first.
---

## Purpose
Define a consistent exception hierarchy so domain and infrastructure errors propagate cleanly to infrastructure boundaries where they are translated into appropriate responses (HTTP status codes, Kafka DLQ, etc.). Keep the happy path linear and free of error-shuttling plumbing.

Default to **let it fail**: treat domain errors as exceptions, let them propagate to a single boundary handler. Only deviate — by switching a specific operation to a sealed result type — when production monitoring shows that one exception is firing often enough to be noise, or measurement shows it's the bottleneck.

Assume callers are internal by default (own UI, internal APIs): a domain error then signals a bug or an unexpected situation, not a normal outcome.

## Strategy
1. Domain layer throws domain-specific exceptions, named for the invariant they broke, to signal business rule violations
2. Application services let domain exceptions propagate (no catch)
3. Infrastructure adapters (controllers, consumers) let exceptions reach the global handler
4. Infrastructure exceptions (DB failures, HTTP client errors) are separate from domain exceptions and defined close to the infrastructure layer
5. The boundary handler (e.g. `GlobalExceptionHandler`, a global exception filter) owns the translation to transport concerns: HTTP status, error code string, log line, metric counter — and sends exactly one response in a consistent shape

## Guidelines

**DO:**
- Apply the Let It Fail principle: do not defensively catch domain exceptions just to log or rethrow — allow them to propagate to the appropriate boundary (e.g., API layer or global handler)
- Create a closed (sealed/abstract) `DomainException` base class for all domain errors
- Make the domain exceptions a method can throw visible in its signature where the language supports it, just for minimal visibility
- Create specific exception subclasses for each business error case, named for the broken invariant (e.g., `InsufficientFundsException`, `AccountNotFoundException`)
- Throw domain exceptions in domain entities or domain services only
- Let exceptions propagate naturally through application services without catching
- Use separate infrastructure exceptions for infra failures (DB, HTTP client, Kafka), extending their own `InfrastructureException` base (e.g., `HttpCallNonSucceededException`, `OptimisticLockingException`)
- Create infrastructure-specific exceptions only when you're adding meaningful context or converting low-level failures into something your application understands (e.g. a unique-violation into `OptimisticLockingException`, an HTTP failure enriched with the client name and upstream status); otherwise let library/framework exceptions propagate instead of wrapping them unnecessarily. When wrapping, keep the original as the cause
- Keep the message specific (`"Insufficient funds: balance=..., required=..."`) — the exception class name + message are the diagnostic signal
- Branch on the exception type, never on its name or message string
- Map exception → HTTP status + API error code (e.g. `"INSUFFICIENT_FUNDS"`) **in the boundary handler**, not on the exception itself
- Give every domain exception exactly one mapping in the boundary handler, added in the same change as the exception. An unmapped exception falls through to 500, so a validation or conflict case silently reports as an internal error
- Increment a metric counter per exception type in the boundary handler — this is the signal that drives later optimization decisions
- Let the framework's built-in request validation produce 400s for transport-level input errors (malformed body, invalid path param); the domain handler is only for domain exceptions

**DON'T:**
- Catch and rethrow domain exceptions in application services - let them propagate
- Use generic exceptions (`RuntimeException`, `IllegalStateException`, plain `Error`) for domain errors - name the broken rule
- Return `null` to signal "rule violated" - throw the named exception
- Throw domain exceptions from infrastructure adapters (repositories, producers, HTTP clients) - use infrastructure exceptions instead
- Use try-catch in controllers - delegate to the global handler
- Mix domain and infrastructure exception hierarchies
- Catch exceptions just to log and rethrow - let them propagate to global handlers for consistent logging and response mapping
- Map domain exceptions to HTTP inside the domain or application layers — that leaks transport concerns inward
- Store HTTP status, API error codes, or other transport concerns on the exception. The exception describes *what went wrong in the domain*; the handler decides how to render it. The same `InsufficientFundsException` may be a 422 over HTTP and a DLQ poison message on Kafka — that's a handler decision

## Monitor first, then promote hot exceptions to a result type

Exceptions are not free — most runtimes capture a stack trace when an exception is constructed, which costs time proportional to stack depth. Rare exceptions don't matter; hot ones can.

**The loop:** monitor → measure → promote

1. **Default**: throw normal exceptions. They're rare, the trace is the primary debugging signal, and the happy path stays linear.
2. **Monitor**: a counter per exception type in the boundary handler (`errors{type=..., useCase=...}`)
3. **Measure**: a specific exception's rate is high enough that (a) it dominates the `errors` counter, (b) it shows up on flame graphs / latency dashboards, *or* (c) the business reclassifies it as an expected outcome (e.g. validation failures on a public-facing endpoint, where rejection is normal, not exceptional).
4. **Promote that operation to a sealed result type**: write a micro-benchmark of the hot operation comparing the throw-based implementation vs. the result-type variant. Don't skip this — if the throw site isn't the bottleneck, changing the signature is churn.

This is intentionally surgical: one operation flips at a time, only when justified by data. The rest of the codebase keeps the readable throw-based style.

### Result type, not Either

For the targeted promotion described above, prefer a per-operation sealed result (e.g. `AddMemberResult` with one variant per outcome) over a generic `Either<Error, Success>`. The result type names the *specific* outcomes of *that* operation, which is what makes the call site readable. If the project has chosen to use `Either` pervasively, see the separate `handling-errors-with-either` skill — the rules above still describe the default exception-based approach.

### When to skip straight to a result type
Skip the "default first, promote later" loop and define a sealed result from the start when:

- **The operation is expected to fail on a substantial share of traffic** — e.g. a credential-check endpoint where wrong-password is normal, or an idempotency-replay check expected to see duplicate keys under retry.
- **You don't control the inputs** — e.g. a public / open API where you don't own the UI doing client-side validation, so callers will send wrong ids, malformed emails, missing fields, etc. as a normal part of usage. Treating "bad input from an external caller" as exceptional is a category error: it's the expected case, and the throw cost compounds at the edge.

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
