---
name: applying-pragmatic-hexagonal
description: ALWAYS invoke this skill when creating, moving or renaming code, packages or folders, when planning (plan mode) where new code should live, when reviewing project structure, or when the user asks about layers or dependency rules - in projects using hexagonal architecture. Covers domain/application/infrastructure layers, ports and adapters, and package or folder layout. For layered projects use applying-pragmatic-layered. Do not decide where code lives directly - use this skill first.
---

## Purpose
Organise code with hexagonal architecture so business is protected from the rippling effects of infrastructure changes.
The following guidelines provide a pragmatic approach to hexagonal architecture that avoids unnecessary abstractions and boilerplate.

## Principles

1. Produce clean, modular, testable hexagonal code.
2. Separate business logic from infrastructure concerns.
3. Use ports and adapters to isolate the core domain from external systems: the domain defines ports, infrastructure implements them.
4. Domain entities hold all business logic and rules; application services orchestrate domain and infrastructure (through ports) but contain no business logic.
5. Maintainability and clean code should be preferred over hexagonal purity.
6. Always place files in the correct layer and path according to the project structure.
7. Take a look always at the hexagonal shortcuts rules to understand which shortcuts are allowed in this project.

## Dependency Rule
`inbound infra adapters → application → domain ← outbound infra adapters`

- Domain layer has NO dependencies on other layers (and no framework dependencies)
- Application layer depends on domain only (we may leak framework annotations/decorators here for pragmatic reasons)
- Inbound infra adapters depend on application and domain
- Outbound infra adapters depend on application and domain ports 

### Dependency Rule Exceptions

- **Infra inbound → Infra outbound (skipping application layer and ports):** when infrastructure needs to communicate with other infrastructure, it can skip ports and use direct communication between them, e.g. a Kafka consumer calling a database repository implementation. This is a pragmatic shortcut that avoids unnecessary indirection when the communication is purely technical and doesn't involve business logic.
- **Infra outbound → Infra outbound:** when one infrastructure component needs to call another (e.g., an HTTP client calling another service), it can depend directly without going through the domain or application layers, as this is purely technical communication.

## Layers
Three top-level layers under the source root, no others:

| Layer | Contains |
|---|---|
| domain | Core business logic (pure, no framework dependencies): aggregates, entities, value objects, domain events, domain errors, ports |
| application | Use cases - orchestration of infrastructure and domain to provide business services: command services (write operations) and query handlers (read operations) |
| infrastructure | Infrastructure adapters, thin translation layer: **inbound** (HTTP controllers, message consumers, schedulers), **outbound** (repositories, HTTP clients, message producers), **config** (framework configuration and wiring) |

Two accepted layouts:
- **DDD style** — `domain` (model + ports), `application` (services/use cases, queries), `infra(structure)` (inbound, outbound, config)
- **By-the-book style** — `domain` holding `model`, `ports` and `usecases`; `infrastructure` (inbound, outbound, config)

Follow the layout the project already uses; the concrete package/folder trees and naming conventions per language are in the examples.

### Structure Violations

- Any folder that does not exist in the structures above is a violation. No extra layers, renamed folders, or intermediate packages/folders at the root are allowed.
- New packages/folders are allowed only on top of existing ones.

### Pragmatic: hexagonal shortcuts
These **shortcuts are allowed**, they intentionally simplify classical hexagonal architecture:
- **No explicit port interfaces for inbound adapters** — controllers, consumers and other inbound adapters call application services directly
- **Framework annotations/decorators in the application layer are acceptable** when they reduce boilerplate without leaking framework logic (DI registration, transaction demarcation)
- **Leak domain to infrastructure** — domain model can be used in inbound and outbound adapters as arguments and return types, we don't need to create separate DTOs or mappers if the domain model is already suitable for that purpose
- **Keep packages/folders flat** — Create sub-packages/sub-folders only when one exceeds 10 files (Don't create `outbound/db`, `outbound/messaging` etc. for a few files)
- **Avoid multi-module projects** - Multi-module projects (Gradle modules, monorepo workspaces) create unnecessary complexity and boilerplate, prefer a single module with a clear structure

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
