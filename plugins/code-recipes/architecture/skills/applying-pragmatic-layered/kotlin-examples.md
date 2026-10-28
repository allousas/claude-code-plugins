# Kotlin Examples

## Standard style
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

## Spring specifics
- Accepted framework annotations in the business layer: `@Service`, `@Transactional`
- Infrastructure classes are Spring beans (`@Repository`, `@Component`) injected directly into services; tests mock them with MockK
