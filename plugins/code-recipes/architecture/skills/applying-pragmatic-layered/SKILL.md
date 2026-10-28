---
name: applying-pragmatic-layered
description: ALWAYS invoke this skill when creating, moving or renaming code, packages or folders, when planning (plan mode) where new code should live, when reviewing project structure, or when the user asks about layers or dependency direction - in projects using layered architecture. Covers entrypoint/business/infrastructure layers without ports, and package or folder layout. For hexagonal projects use applying-pragmatic-hexagonal. Do not decide where code lives directly - use this skill first.
---

## Purpose
Organise code with a modern layered architecture so business logic is clearly separated from infrastructure concerns.
The following guidelines provide a pragmatic approach to layered architecture that keeps the simplicity of traditional 3-layer with a modern approach.

## Principles

1. Produce clean, modular, testable layered code.
2. Separate business logic from infrastructure concerns through package/folder boundaries, not abstractions.
3. No port interfaces — business services depend on concrete infrastructure classes directly.
4. Maintainability and simplicity should be preferred over architectural purity.
5. Always place files in the correct layer and path according to the project structure.
6. Take a look always at the layered shortcuts rules to understand which shortcuts are allowed in this project.

## Dependency Rule
`entrypoint → business → infrastructure`

- Business layer depends on infrastructure (traditional downward direction, no dependency inversion)
- Entrypoint layer depends on business and infrastructure
- Infrastructure has no dependencies on other layers

## Layers

| Layer | Contains |
|---|---|
| business/domain | Aggregates, entities, value objects, domain events, domain errors |
| business/services | Use cases — orchestration of infrastructure and domain |
| entrypoint | Inbound adapters — triggers that invoke business services (HTTP endpoints, Kafka consumers, etc.) |
| infrastructure | Outbound adapters — technical concerns: db (repositories), clients (internal and external HTTP clients), observability (logging, metrics via domain event listeners), config (framework configuration and wiring) |

The concrete package/folder trees and naming conventions per language are in the examples.
Allow any other structure as long as it is consistent and logical to layered architecture.

### Layered shortcuts
This approach intentionally simplifies classical layered architecture:
- **No port interfaces** — business services depend on concrete infrastructure classes, no need for interfaces when there is a single implementation
- **Framework annotations/decorators in the business layer are acceptable** when they reduce boilerplate without leaking framework logic (DI registration, transaction demarcation)
- **Leak domain to entrypoints and infrastructure** — domain model can be used directly in other layers, we don't need to create separate DTOs for translation if the domain model is already suitable for that purpose
- **Avoid multi-module projects** - Multi-module projects (Gradle modules, monorepo workspaces) create unnecessary complexity and boilerplate, prefer a single module with a clear structure

## How It Differs from Hexagonal

| Aspect | Hexagonal | Modern Layered |
|---|---|---|
| Port interfaces | Domain defines interfaces for all outbound dependencies | No interfaces — services depend on concrete infrastructure classes |
| Dependency direction | Infrastructure depends on domain (dependency inversion) | Business depends on infrastructure (traditional downward) |
| Adapter registration | Adapters implement domain-defined ports | Infrastructure classes are registered in the DI container and injected directly |
| Testability | Test domain with fake implementations of ports | Tests use mocking libraries to mock concrete infrastructure classes |
| Complexity | Higher ceremony (interface per dependency) | Lower ceremony (no boilerplate interfaces) |

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
