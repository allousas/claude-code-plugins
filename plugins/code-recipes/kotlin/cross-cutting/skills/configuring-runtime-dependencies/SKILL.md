---
name: configuring-runtime-dependencies
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing any file that is a Spring configuration class, bean definition, dependency injection wiring, or configuration properties binding.
---

## Purpose
Framework configuration and runtime wiring of the different components of the application.

## Guidelines

**DO:**
- Use constructor-based injection everywhere - no field injection, no setter injection
- Use separate configuration classes with programmatic wiring for complex dependencies or when auto-wiring is not enough
- Group related bean definitions in dedicated configuration classes (e.g., `KafkaConfig`, `HttpClientConfig`)
- Keep configuration classes in the infrastructure layer
- Use default parameter values in constructors for optional dependencies (e.g., `generateId: () -> UUID = { UUID.randomUUID() }`)

**DON'T:**
- Put configuration classes in domain or application layers - they belong in the infrastructure layer
- Create circular dependencies between beans

### Spring specifics
- `@Configuration` classes declare `@Bean` methods to create and wire infrastructure components
- `@ConfigurationProperties(prefix = "...")` binds YAML/properties to a data class
- Don't use `@Autowired` annotation. Use constructor-based injection instead
- Use spring starter modules to avoid manual configuration of common dependencies (e.g., spring-boot-starter-web, spring-boot-starter-data-jdbc, spring-boot-starter-kafka)
- Use `@Value` for simple configurations - use `@ConfigurationProperties` for complex configurations with multiple related properties

## Examples
Please use always these examples as reference: [examples.md](examples.md)
