# Kotlin Examples

## Default Tech Stack

| Category | Default |
|----------|---------|
| Language | Kotlin (latest) |
| Framework | Spring Boot 4 |
| Database | PostgreSQL |
| DB Access | Spring JDBC |
| Migrations | Flyway |
| Build Tool | Gradle (latest) with `libs.versions.toml` |
| JSON | Jackson |
| REST Clients | Retrofit |
| Test Framework | JUnit 5 |
| Mocking | Mockk |
| Assertions | Kotest |
| Containers | TestContainers |
| HTTP Stubs | WireMock |
| API Testing | RestAssured |

## Default Architecture (hexagonal, DDD style)
```
src/main/kotlin/<base-package>/
├── domain/model/
├── application/services/
└── infra/inbound/ | outbound/ | config/
```

## Required Input
- **Service name** (e.g. `team-service`)
- **Base package** (e.g. `com.company.teamservice`)

## Spring / Gradle specifics
- Single Gradle module; all dependency versions in `gradle/libs.versions.toml`
- Flyway migrations in `src/main/resources/db/migration`
