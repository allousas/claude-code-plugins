---
name: versioning-database-schema
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing any Flyway migration file, SQL schema change, or database table modification. This includes adding, modifying, or fixing migration scripts (V*__.sql), CREATE TABLE statements, ALTER TABLE statements, index creation, column additions, or db/migration/ files.
---

## Purpose
Version-control database schema changes with Flyway so that every environment (local, CI, staging, production) converges to the same schema state reliably and repeatably.

## Principles

1. Every schema change is a migration — never modify the database by hand.
2. Migrations are immutable — once applied, never edit them; create a new migration instead.
3. Migrations are forward-only — avoid rollback scripts; fix forward with a new migration.
4. Each migration does one thing — don't mix unrelated table changes in a single file.
5. Migrations live in the repository alongside the code that depends on them.

## Migration File Conventions

### Naming
```
V<NNN>__<description>.sql
```
- `V` prefix for versioned migrations (executed once, in order)
- `<NNN>` — zero-padded sequential number: `V001`, `V002`, ..., `V042`
- `__` — double underscore separator
- `<description>` — snake_case summary of the change: `create_team_table`, `add_version_to_annual_leave`

### Location
```
src/main/resources/db/migration/
├── V001__create_team_table.sql
├── V002__create_outbox_table.sql
├── V003__add_version_to_annual_leave.sql
└── V004__add_index_on_team_name.sql
```

## Schema Conventions

- Always add `created_at TIMESTAMP NOT NULL DEFAULT now()` and `updated_at TIMESTAMP NOT NULL DEFAULT now()` to all tables
- Use `UUID` for primary keys
- Use `snake_case` for all table and column names
- Use plural table names (e.g., `teams`, `annual_leaves`) for consistency
- Add a `version INTEGER NOT NULL DEFAULT 0` column to entities that require optimistic locking
- Prefer `JSONB` over multiple join tables for nested/variable structures when querying flexibility is needed
- Add indexes explicitly — don't rely on implicit index creation except for primary keys
- Use `NOT NULL` by default — only allow `NULL` when the business domain requires it

## Guidelines

**DO:**
- Create one migration per logical change (new table, new column, new index)
- Test migrations against a real database (TestContainers) in integration and component tests
- Include `IF NOT EXISTS` / `IF EXISTS` guards in DDL when appropriate for safety
- Add comments in SQL for non-obvious decisions (e.g., why a partial index, why a specific default)
- Keep migrations small and focused — easier to review and debug

**DON'T:**
- Edit or delete a migration that has already been applied (Flyway checksum validation will fail)
- Use `DROP TABLE` or `DROP COLUMN` without confirming the data is no longer needed
- Add business logic in migrations (data transformations, conditional inserts based on business rules)
- Use database-specific features that prevent portability unless the project is committed to a single database
- Create migrations that lock large tables for long periods in production — prefer non-blocking operations (e.g., `CREATE INDEX CONCURRENTLY` in PostgreSQL)

### Spring specifics
- Spring Boot auto-configures Flyway when `org.flywaydb:flyway-core` is on the classpath
- Migrations run automatically on application startup from `src/main/resources/db/migration/`
- Use `spring.flyway.*` properties for configuration (url, user, password, locations)
- In tests, Flyway runs against the TestContainers database via the same migration files

## Examples
Please use always these examples as reference: [examples.md](examples.md)
