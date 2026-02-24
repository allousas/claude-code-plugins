---
name: setting-up-a-new-project
description: Apply when bootstrapping a new microservice project from scratch. Always confirm the default tech stack and architecture with the user before generating any code.
---

## Purpose
Bootstrap a new microservice project with the correct structure, dependencies, configuration, and conventions.

## Before You Start — Confirm Defaults

**CRITICAL: Before generating any code, present the defaults below and ask the user to confirm or override.**

Present this summary and ask: _"I'll set up the project with these defaults. OK, or do you want to change anything?"_

### Default Tech Stack 

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

### Default Architecture
Hexagonal (DDD style):
```
domain/model/
application/services/
infra/inbound/ | outbound/ | config/
```

### Required Input
Only ask the user for:
- **Service name** (e.g. `team-service`)
- **Base package** (e.g. `com.company.teamservice`)

Everything else uses defaults unless the user says otherwise.

## Project Generation 

After confirmation, generate the project.