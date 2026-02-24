---
name: applying-pragmatic-layered
description: Apply when creating or moving, files or packages (also plan mode) and project is using layered architecture.
---

## Purpose
Organise code with a modern layered architecture so business logic is clearly separated from infrastructure concerns.
The following guidelines provide a pragmatic approach to layered architecture that keeps the simplicity of traditional 3-layer with a modern approach.

## Principles

1. Produce clean, modular, testable layered code.
2. Separate business logic from infrastructure concerns through package boundaries, not abstractions.
3. No port interfaces — business services depend on concrete infrastructure classes directly.
4. Maintainability and simplicity should be preferred over architectural purity.
5. Always place files in the correct layer and path according to the project structure.
6. Take a look always at the layered shortcuts rules to understand which shortcuts are allowed in this project.

## Dependency Rule
```
entrypoint → business → infrastructure
```

- Business layer depends on infrastructure (traditional downward direction, no dependency inversion)
- Entrypoint layer depends on business and infrastructure
- Infrastructure has no dependencies on other layers

## Package Structure

### Standard style
```
src/main/kotlin/<company-package>/
├── business/           # Core business logic
│   ├── domain/         # Aggregates, entities, value objects, domain events, domain errors
│   └── services/       # Use cases — orchestration of infrastructure and domain
├── entrypoint/         # Inbound adapters — triggers that invoke business services (HTTP endpoints, Kafka consumers, etc...)
└── infrastructure/     # Outbound adapters — technical concerns
    ├── db/             # Repositories
    ├── clients/        # HTTP clients (internal and external)
    ├── observability/  # Logging, metrics via domain event listeners
    └── config/         # Spring configuration, framework wiring
```

Allow any other structure as long as it is consistent and logical to layered architecture.

### Layered shortcuts
This approach intentionally simplifies classical layered architecture:
- **No port interfaces** — business services depend on concrete infrastructure classes, no need for interfaces when there is a single implementation
- **Framework annotations in the business layer are acceptable** when they reduce boilerplate without leaking framework logic (`@Transactional`, `@Service`)
- **Leak domain to entrypoints and infrastructure** — domain model can be used directly in other layers, we don't need to create separate DTOs for translation if the domain model is already suitable for that purpose
- **Avoid multimodule projects** - Multimodule projects create unnecessary complexity and boilerplate, prefer a single module with clear package structure instead of splitting into multiple modules

## How It Differs from Hexagonal

| Aspect | Hexagonal | Modern Layered |
|---|---|---|
| Port interfaces | Domain defines interfaces for all outbound dependencies | No interfaces — services depend on concrete infrastructure classes |
| Dependency direction | Infrastructure depends on domain (dependency inversion) | Business depends on infrastructure (traditional downward) |
| Adapter registration | Adapters implement domain-defined ports | Infrastructure classes are Spring beans injected directly |
| Testability | Test domain with fake implementations of ports | Tests use mocking libraries to mock concrete infrastructure classes |
| Complexity | Higher ceremony (interface per dependency) | Lower ceremony (no boilerplate interfaces) |
