---
name: default-tech-stack
description: Default technology stack for microservices. Kotlin, Spring Boot 4, PostgreSQL, and related tooling. (project)
---

# Default Tech Stack

This skill defines the default technology stack for microservices in this project.

## Core Stack

| Category | Technology |
|----------|------------|
| Language | Kotlin (latest) |
| Framework | Spring Boot 4 |
| Database | PostgreSQL |
| DB Access | Spring JDBC |
| Migrations | Flyway |
| Build Tool | Gradle (latest) |
| Dependency Versions | libs.versions.toml |

## HTTP & APIs

| Category | Technology |
|----------|------------|
| REST Clients | Retrofit |
| JSON | Jackson |

## Testing

| Category | Technology |
|----------|------------|
| Test Framework | JUnit 5 |
| Mocking | Mockk |
| Assertions | Kotest |
| Containers | TestContainers |
| HTTP Stubs | WireMock |
| API Testing | RestAssured |

## Gradle Plugins

- `kotlin-spring` - Automatically opens Spring-annotated classes (never use `open` keyword manually)
- `kotlin-jvm` - Kotlin JVM support
- `org.springframework.boot` - Spring Boot plugin
- `org.flywaydb.flyway` - Database migrations

## Configuration

- Use `application.yml` for configuration
- Use `@ConfigurationProperties` for type-safe config
- Environment-specific configs: `application-{profile}.yml`
