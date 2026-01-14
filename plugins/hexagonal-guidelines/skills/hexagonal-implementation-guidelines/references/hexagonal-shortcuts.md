# Hexagonal Architecture Shortcuts

This project follows Hexagonal Architecture (Ports & Adapters) principles with some architectural shortcuts for simplicity and pragmatism.

## Shortcuts Applied

### 1. No Inbound Ports
- **Rule**: Inbound adapters call application services directly
- **This Project**: Only outbound ports, no need to create interface for them, skip them to reduce boilerplate

### 2. Direct Outbound Access (when appropriate)
- **Rule**: Inbound adapters can call outbound adapters directly when no business logic is needed
- **Use Cases**: Simple data retrieval, data replication, queries without business rules
- **Example**: A simple GET endpoint that just fetches data from repository

### 3. Domain Layer Leaking (Controlled)
- **Rule**: Domain layer can be leaked in infrastructure layers, but not vice versa
- **Allowed Leaking**: Domain entities, domain events, domain exceptions/errors can be used in infrastructure
- **Not Allowed**: Infrastructure concerns (frameworks, libraries) cannot leak into domain

### 4. Framework Annotations in Application Services
- **Allowed Shortcuts**:
  - `@Transactional` - Can be used in application services for transaction management
  - `@Service` - Can be used in application services for Spring DI
- **Rationale**: Pragmatic trade-off between purity and simplicity
- **Boundary**: Don't leak other framework concerns into domain layer

## What NOT to Shortcut

❌ **Don't violate these principles:**
- Keep domain layer pure (no framework dependencies except @Transactional and @Service in app services)
- Don't put business logic in infrastructure layers
- Don't create dependencies between application services
- Don't skip outbound ports - always define port interfaces in domain layer
- Don't mix different aggregates in a single application service

## Summary
Maintainability and clean code should be preferred over hexagonal purity, but don't compromise core hexagonal principles (separation of concerns, dependency direction, testability).