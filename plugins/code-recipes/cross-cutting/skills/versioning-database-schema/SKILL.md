---
name: versioning-database-schema
description: Invoke when writing or reviewing database migrations or any schema change (including plans), including when a feature needs a new table or column, or when the user asks about evolving the schema. Covers migration scripts, tables, columns, indexes and constraints.
---

## Purpose
Version-control database schema changes with a migration tool so that every environment (local, CI, staging, production) converges to the same schema state reliably and repeatably.

## Principles

1. Every schema change is a migration — never modify the database by hand.
2. Migrations are immutable — once applied, never edit them; create a new migration instead.
3. Migrations are forward-only — avoid rollback scripts; fix forward with a new migration.
4. Each migration does one thing — don't mix unrelated table changes in a single file.
5. Migrations live in the repository alongside the code that depends on them.

## Migration File Conventions

- Follow the migration tool's file naming convention; the prefix (sequential number or timestamp) must sort in apply order
- Describe the change in the file name in `snake_case`: `create_teams_table`, `add_version_to_annual_leaves`
- Keep all migrations in the single directory the tool reads from; generate new files with the tool's own command when it has one

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
- Test migrations against a real database (containerized) in integration and component tests, applied by the same tool and files as production
- Include `IF NOT EXISTS` / `IF EXISTS` guards in DDL when appropriate for safety
- Add comments in SQL for non-obvious decisions (e.g., why a partial index, why a specific default)
- Keep migrations small and focused — easier to review and debug

**DON'T:**
- Edit or delete a migration that has already been applied (tools record applied migrations, and checksum-validating ones fail; others silently skip the edit)
- Use `DROP TABLE` or `DROP COLUMN` without confirming the data is no longer needed
- Add business logic in migrations (data transformations, conditional inserts based on business rules)
- Use database-specific features that prevent portability unless the project is committed to a single database
- Create migrations that lock large tables for long periods in production — prefer non-blocking operations (e.g., `CREATE INDEX CONCURRENTLY` in PostgreSQL, which must run outside a transaction)

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
