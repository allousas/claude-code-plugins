# Kotlin Examples

## DDD style
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

## By-the-book style
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

## Spring specifics
- Accepted framework annotations in the application layer: `@Service`, `@Transactional`
- Ports are Kotlin `interface`s in the domain; adapters implement them and are registered as Spring beans (`@Repository`, `@Component`) or in `config/`
- Single Gradle module — no `domain`/`application`/`infra` Gradle subprojects
