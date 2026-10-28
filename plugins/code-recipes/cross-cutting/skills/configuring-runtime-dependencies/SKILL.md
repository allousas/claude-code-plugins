---
name: configuring-runtime-dependencies
description: ALWAYS invoke this skill when writing, changing, refactoring, planning (plan mode) or reviewing framework configuration or dependency injection wiring, or when the user asks about DI or configuration. Covers configuration classes and modules, bean or provider definitions, binding ports to adapters, injection tokens, configuration properties and environment variables, and global filters or interceptors. Do not wire modules or configure DI directly - use this skill first.
---

## Purpose
Framework configuration and runtime wiring of the different components of the application. This is the composition root: domain ports are bound to their infrastructure adapters, application services are registered, and global cross-cutting concerns (validation, exception mapping) are configured. Framework wiring lives here and nowhere else.

## Flow
1. Define ports in the domain layer (no framework types)
2. Implement adapters in the infrastructure layer
3. In configuration under the infrastructure layer, bind each port to its adapter
4. Register application services so their dependencies are resolved by DI
5. Register global cross-cutting providers (validation, error mapping) once, at the composition root

## Guidelines

**DO:**
- Use constructor-based injection everywhere - no field injection, no setter injection
- Use separate configuration classes/modules with programmatic wiring for complex dependencies or when auto-wiring is not enough
- Group related definitions in dedicated configuration units (e.g., `KafkaConfig`, `HttpClientConfig`, `database.providers`)
- Keep configuration in the infrastructure layer
- Bind each outbound port to exactly one adapter
- Register cross-cutting concerns (validation, error mapping) once, globally
- Keep configuration declarative — a list of components, providers and bindings
- Make sources of non-determinism (id generation, clock) injectable dependencies with a production default, so tests can override them
- Bind groups of related configuration properties to typed configuration objects; use single-value lookups only for simple settings

**DON'T:**
- Put configuration in domain or application layers - it belongs in the infrastructure layer
- Create circular dependencies between components
- Put business logic, orchestration, or mapping in configuration
- Instantiate adapters inside application services or controllers - let DI construct them
- Register the same provider in multiple places

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
