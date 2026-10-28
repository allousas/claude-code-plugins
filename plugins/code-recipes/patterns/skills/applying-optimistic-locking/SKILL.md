---
name: applying-optimistic-locking
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing repository methods that save entities with optimistic locking - version fields/columns, version checks in UPDATE or upsert statements, OptimisticLockingException, concurrent modification and lost updates (JDBC, Kysely or any SQL access).
---

## Purpose
Prevent lost updates when multiple processes modify the same entity concurrently. Each entity carries a version field that is checked during updates - if the version has changed since the entity was read, the update is rejected.

## Typical Flow
1. Entity includes a `version` field (integer, starting at 0)
2. Repository reads entity with current version
3. Application service performs domain operations on the entity; the new state carries the version it was loaded from, unchanged
4. Repository saves entity with `WHERE version = ?` condition, incrementing the version in the same statement
5. If no rows are updated (version mismatch), throw `OptimisticLockingException`
6. Caller decides retry strategy (controller retries, consumer relies on broker redelivery/retry)

## Guidelines

**DO:**
- Add an integer `version` field to entities that can be concurrently modified
- Initialise the version to 0 when creating a new entity
- Use `WHERE version = ?` in UPDATE/INSERT ON CONFLICT SQL statements
- Throw a specific `OptimisticLockingException` when zero rows are affected
- Increment version in the SQL statement itself (`version = version + 1` or `version = <table>.version + 1`)
- Keep the version check inside the repository - callers should not manage versions
- Define `OptimisticLockingException` as an infrastructure exception (not a domain one), keeping the driver error as `cause` when wrapping one
- Translate a unique-key violation when inserting a new entity into `OptimisticLockingException` (a concurrent creation)
- Test optimistic locking against the real database engine with genuine commits - not mocks, not rolled-back-transaction isolation

**DON'T:**
- Increment the version in application or domain code - let the database handle it
- Silently ignore zero affected rows - always throw to signal the conflict
- Use optimistic locking on entities that are rarely modified concurrently (unnecessary complexity)
- Retry indefinitely - set a maximum retry count

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
