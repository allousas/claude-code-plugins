---
name: applying-pragmatic-hexagonal
description: Apply when creating or moving, files or packages (also plan mode) and project is using hexagonal architecture.
---

## Purpose
Organise code with hexagonal architecture so business is protected from the rippling effects of infrastructure changes.
The following guidelines provide a pragmatic approach to hexagonal architecture that avoids unnecessary abstractions and boilerplate.

## Principles

1. Produce clean, modular, testable hexagonal code.
2. Separate business logic from infrastructure concerns.
3. Use ports and adapters to isolate the core domain from external systems.
4. Maintainability and clean code should be preferred over hexagonal purity.
5. Always place files in the correct layer and path according to the project structure.
6. Take a look always at the hexagonal shortcuts rules to understand which shortcuts are allowed in this project.

## Dependency Rule
```
inbound infra adapters → application → domain ← outbound infra adapters
```

- Domain layer has NO dependencies on other layers
- Application layer depends on domain only (we may leak framework annotations here for pragmatic reasons)
- Inbound infra adapters depend on application and domain
- Outbound infra adapters depend on application and domain ports 

### Dependency Rule Exceptions

- **Infra inbound → Infra outbound (skipping application layer and ports):** when infrastructure needs to communicate with other infrastructure, it can skip ports and use direct communication between them, e.g. a consumer of kafka calling repository impl of a database. This is a pragmatic shortcut that avoids unnecessary indirection when the communication is purely technical and doesn't involve business logic.
- **Infra outbound → Infra outbound:** when one infrastructure component needs to call another (e.g., an HTTP client calling another service), it can depend directly without going through the domain or application layers, as this is purely technical communication.

## package Structure

### DDD style
```
src/main/kotlin/<company-package>/
├── domain/model/     # Core business logic (pure, no framework dependencies), Aggregates, entities, value objects, domain events, ports
├── application/      # Use cases - orchestration of infrastructure and domain to provide business services
│   ├── services/     # Command services (write operations)
│   └── queries/      # Query handlers (read operations)
└── infra/            # Infrastructure adapters, thin translation layer
    ├── inbound/      # HTTP controllers, message consumers, schedulers
    ├── outbound/     # Repositories, HTTP clients, message producers
    └── config/       # Spring configuration
```
### By-the-book style
```
src/main/kotlin/<company-package>/
├── domain/      
│   ├── model/       # Core business logic (pure, no framework dependencies), Aggregates, entities, value objects, domain events, ports
│   ├── ports/       # Interfaces for outbound dependencies (repositories, external services)
│   └── usecases/    # Use cases - orchestration of infrastructure and domain to provide business services
└── infrastructure/  # Infrastructure adapters, thin translation layer
    ├── inbound/     # HTTP controllers, message consumers, schedulers
    ├── outbound/    # Repositories, HTTP clients, message producers
    └── config/      # Spring configuration
```

### Package Structure Violations

- Any folder that does not exist in the structures above is a violation. No extra layers, renamed folders, or intermediate packages are allowed.
- New packages are allowed only on top of existing packages.

### Pragmatic: hexagonal shortcuts
These **shortcuts are allowed**, they intentionally simplify classical hexagonal architecture:
- **No explicit port interfaces for inbound adapters** — controllers and other inbound adapters call application services directly
- **Framework annotations in the application layer are acceptable** when they reduce boilerplate without leaking framework logic (`@Transactional`, `@Service`)
- **Leak domain to infrastructure** — domain model can be used in inbound and outbound adapters, we don't need to create separate DTOs for translation if the domain model is already suitable for that purpose
- **Keep Packages Flat** — Create subpackages only when a package exceeds 10 files (Don't create /outbound/db, /outbound/messaging etc... if a few files) 
- **Avoid multimodule projects** - Multimodule projects create unnecessary complexity and boilerplate, prefer a single module with clear package structure instead of splitting into multiple modules
