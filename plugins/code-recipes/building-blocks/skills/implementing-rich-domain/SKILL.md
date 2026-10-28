---
name: implementing-rich-domain
description: Invoke when writing or reviewing domain layer code (including plans), or when the user asks about domain modelling or where business logic belongs. Covers entities, value objects, aggregates, named factories, invariants, domain errors and domain events.
---

## Purpose
The domain layer is the heart of the system. It owns the business rules and the state they protect. A *rich* domain model keeps state and behavior together so invariants cannot be bypassed by callers, and so application services stay free of business logic.

## Building blocks
- **Entity** - has identity and a lifecycle. Equality by id. Behavior changes its state through methods.
- **Value Object** - no identity. Equality by value. Immutable. Used to make implicit concepts explicit.
- **Aggregate** - a cluster of entities/value objects with one root entity as the only entry point. The root enforces invariants for the whole cluster and is the unit of persistence.
- **Domain Error** - typed error/exception expressing a violated invariant, thrown or returned by the entity. Never generic errors/exceptions.
- **Domain Event** - past-tense fact that something happened (e.g. `TeamCreated`).
- **Factory** - a function or class creating complex entities or aggregates when creation needs infrastructure or other aggregates. It lives in the application layer, not the domain, to avoid coupling the domain to infrastructure.

## Flow
1. An entity enters existence either freshly created through a constructor/factory that validates its inputs, or rehydrated from persistence
2. The caller invokes a verb method naming the business action (`addMember`, `lock`)
3. That method checks the invariant guarding the action
4. On violation it throws (or returns) a typed domain error; otherwise it returns a **new instance** carrying the new state
5. The application service decides what to do with that instance - persist it, publish the resulting event

The entity never reaches outward in step 5. It reports what happened; the application service acts on it.

## Guidelines

**DO:**
- Code around aggregates: all business state mutations, invariant checks and domain rules go through the aggregate root, using explicit action verbs
- Put business rules **inside** the entity method that performs the action - never in the application service, controller or repository
- Make state changes return a **new instance**; enforce immutability with read-only fields
- Validate inputs in constructors / factory methods so an instance cannot exist in an invalid state
- Keep creation (validates) and rehydration from persistence (trusts stored data) as separate named entry points - re-validating on every read means a rule change makes stored rows unloadable
- Throw or return a **typed domain error** when an invariant is violated, named after the rule broken (`AddMemberException(DUPLICATE_MEMBER)`, `TeamValidationException(TOO_LONG_NAME)`) - see `cross-cutting:handling-errors-with-exceptions` / `cross-cutting:handling-errors-with-either`
- Use **value objects** only to wrap primitives that carry rules (a 4-digit `Code`, a bounded `TeamName`). This pushes validation to the type and makes signatures self-documenting
- Express the action as a verb method (`addMember`, `lock`) - the method name is the ubiquitous language
- Model lifecycle states as a closed set of values so invalid states are unrepresentable
- Keep the domain layer **dependency-free**: no framework, database, messaging or logging libraries, no system clock or random id calls scattered around (inject clocks/id-generators as parameters when you need them)

**DON'T:**
- Create value objects that aren't needed - don't wrap primitives for the sake of wrapping. If a string is sufficient, use a string
- Expose setters or public mutable fields - they let callers bypass invariants
- Use infrastructure types/dependencies (DB rows, HTTP DTOs, Kafka messages) inside the domain
- Call repositories or event publishers from inside an entity. The entity decides *what happened*; the application service decides *what to do about it* (persist, publish)
- Use framework annotations/decorators. The domain is free of infrastructure and never instantiated by the DI container
- Return `null` to signal "rule violated" - throw/return a typed domain error so the failure is named
- Publish generic events like `TeamUpdated` that don't express the business meaning. If the event is important enough to publish, it's important enough to name

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
