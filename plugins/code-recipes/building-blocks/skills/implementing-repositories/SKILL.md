---
name: implementing-repositories
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing database repositories, data access layers, persistence and entity mapping (classes named or implementing `*Repository`, or abstracting access to a database or external storage).
---

## Purpose
Database repositories translate domain calls into database operations, handling persistence and retrieval of domain entities.

## Typical Flow
1. Receive domain object or query parameters from application service
2. Map domain objects to database DTOs/parameters
3. Execute SQL query
4. Map database result rows back to domain entities
5. Return domain entity, null or nothing depending on the operation

## Guidelines

**DO:**
- Follow the repository pattern: https://martinfowler.com/eaaCatalog/repository.html - always load and save complete domain entities (aggregates), never partial reads or updates
- Use simple string queries or a lightweight query builder if no complex query building logic is needed
- Use lightweight database access libraries instead of full ORM frameworks
- File naming: `<ExternalSystem><Concern>Repository`, e.g., `PostgresTeamsRepository`
- Keep database schema simple - avoid complex joins, views, stored procedures
- For complex reads, prefer dedicated projections/read models (e.g. built from domain events) in a separate repository instead of querying operational tables
- Always add `created_at` and `updated_at` timestamps to all tables
- Use parameterized queries or prepared statements to prevent SQL injection
- Declare DTOs/row types in the same file where they are used and keep them private to it
- Use private functions for simple mapping, dedicated mapper classes for complex transformations
- Keep one repository per entity
- Return null (absent) for lookups that miss, so absence is explicit in the signature
- Use optimistic locking on updates: check the loaded version, advance it in the repository and throw an infrastructure exception (e.g. `OptimisticLockingException`) when it is outdated - see `patterns:applying-optimistic-locking`

**DON'T:**
- Have granular field methods to save or update specific fields of an entity - save the whole entity, if partial updates are needed, then it's a sign that the entity is not well modeled and should be split into multiple entities
- Leak DB dtos OUT of repositories, always map to domain objects, IN and OUT
- Use **ORM** frameworks unless necessary, they are adding an unnecessary complex layer
- Declare transactions here - transactions are declared in application services layer (repositories only join the current one)
- Include business logic - only translation, mapping, and external system integration
- Push complexity to the database - push it to the domain layer instead
- Throw domain exceptions or return domain errors - those belong in domain/application layers

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
