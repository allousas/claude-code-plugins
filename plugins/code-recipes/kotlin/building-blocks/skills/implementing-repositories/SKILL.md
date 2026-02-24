---
name: implementing-repositories
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing database repositories.
---

## Purpose
Database repositories translate domain calls into database operations, handling persistence and retrieval of domain entities.

## Typical Flow
1. Receive domain object or query parameters from application service
2. Map domain objects to database DTOs/parameters
3. Execute SQL query
4. Map database result rows back to domain entities
5. Return domain entity, null or Unit depending on the operation

## Guidelines

**DO:**
- Follow the repository pattern: https://martinfowler.com/eaaCatalog/repository.html
- Use simple string queries if no complex query building logic is needed
- Use lightweight database access libraries, e.g., JDBCTemplate, instead of full ORM frameworks
- File naming: `<ExternalSystem><Concern>Repository`, e.g., `PostgresTeamsRepository`
- Keep database schema simple - avoid complex joins, views, stored procedures
- Always add `created_at` and `updated_at` timestamps to all tables
- Use parameterized queries or prepared statements to prevent SQL injection
- Declare DTOs in the same file where they are used for cohesion
- Use internal private functions for simple mapping, dedicated mapper classes for complex transformations
- Keep one repository per entity

**DON'T:**
- Have granular field methods to save or update specific fields of an entity - save the whole entity, if partial updates are needed, then it's a sign that the entity is not well modeled and should be split into multiple entities
- Leak DB dtos OUT of repositories, always map to domain objects, IN and OUT
- Use **ORM** frameworks unless necessary, they are adding an innecessary complex layer
- Declare transactions here - transactions are declared in application services layer
- Include business logic - only translation, mapping, and external system integration
- Push complexity to the database - push it to the domain layer instead
- Throw domain exceptions or return domain errors - those belong in domain/application layers

### Spring specific
- Use `@Repository` annotation
- Use JDBCTemplate for database access

## Examples
Please use always these examples as reference: [examples.md](examples.md)