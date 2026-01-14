---
name: hexagonal-implementation-guidelines
description: Apply Hexagonal Architecture (Ports & Adapters) with Domain-Driven Design principles style. Use when writing or reviewing Kotlin/Spring Boot microservice code. (project)
---

# Hexagonal Architecture with DDD Style

This skill provides our company's coding guidelines and architectural guidelines for building microservices following Hexagonal Architecture (Ports & Adapters) with Domain-Driven Design principles.

## When to Use

This skill is automatically applied when:
- Writing hexagonal with ddd code
- Reviewing code for architectural compliance
- Making decisions about where to place code
- Understanding the project's coding conventions

## Architecture Overview

## 🏛 Hexagonal Architecture Principles

Goals:
1. Produce clean, modular, testable hexagonal code.
2. Separate business logic from infrastructure concerns.
3. Use ports and adapters to isolate the core domain from external systems.
4. Maintainability and clean code should be preferred over hexagonal purity.
5. Use **constructor-based dependency injection**.
6. Always place files in the correct layer and path according to the project structure.
7. Follow the coding guidelines and rules strictly.
8. Take a look always at the hexagonal shortcuts rules to understand which shortcuts are allowed in this project.

### Layer Structure

```
src/main/kotlin/<domain>/
├── domain/           # Core business logic (pure, no framework dependencies)
├── application/      # Use cases and orchestration
│   ├── services/     # Command services (write operations)
│   └── queries/      # Query handlers (read operations)
└── infra/            # Infrastructure adapters, thin translation layer
    ├── inbound/      # HTTP controllers, message consumers, schedulers
    ├── outbound/     # Repositories, HTTP clients, message producers
    └── config/       # Spring configuration
```

### Dependency Direction

```
inbound → application → domain ← outbound
```

- Domain layer has NO dependencies on other layers
- Application layer depends on domain only
- Infrastructure adapters depend on domain and application

## Reference Guidelines (Lazy Loading)

**CRITICAL: Do NOT load all references upfront. Load only what you need when you need it, when working on that layer or concern.**

### General guidelines 
When to use:

- `references/code-conventions.md` - For general coding conventions, check them always
- `references/error-handling.md` - When throwing exceptions and handling errors
- `references/hexagonal-shortcuts.md` - Pragmatic shortcuts allowed, breaking hexagonal purity
- `references/side-effects.md` - Where side effects should occur
- `references/test-guidelines.md` - Testing guidelines and pyramid
- `references/tech-stack.md` - Tech stack

### Layer-Specific guidelines

Check them always when coding in a specific layer:

- `references/layers/domain-layer.md` - Rules for coding Aggregates, entities, value objects, ports
- `references/layers/app-services-layer.md` - Rules for coding Command and query services
- `references/layers/infra-inbound.md` - Controllers, consumers, schedulers ...
- `references/layers/infra-outbound.md` - Repositories, HTTP clients, publishers ...
- `references/layers/infra-config.md` - Spring configuration

