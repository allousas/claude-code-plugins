---
name: applying-optimistic-locking
description: Apply when creating, refactoring, changing, planning (plan mode) or reviewing repository methods that save entities with optimistic locking.
---

## Purpose
Prevent lost updates when multiple processes modify the same entity concurrently. Each entity carries a version field that is checked during updates - if the version has changed since the entity was read, the update is rejected.

## Typical Flow
1. Entity includes a `version` field (integer, starting at 0)
2. Repository reads entity with current version
3. Application service performs domain operations on the entity
4. Repository saves entity with `WHERE version = ?` condition
5. If no rows are updated (version mismatch), throw `OptimisticLockingException`
6. Caller decides retry strategy (controller retries, consumer relies on Kafka retry)

## Guidelines

**DO:**
- Add a `version: Long` field to entities that can be concurrently modified
- Let version increment to 0 automatically when creating a new entity
- Use `WHERE version = ?` in UPDATE/INSERT ON CONFLICT SQL statements
- Throw a specific `OptimisticLockingException` when zero rows are affected
- Increment version in the SQL statement itself (`version = version + 1` or `version = EXCLUDED.version + 1`)
- Keep the version check inside the repository - callers should not manage versions

**DON'T:**
- Increment the version in application code - let the database handle it
- Silently ignore zero affected rows - always throw to signal the conflict
- Use optimistic locking on entities that are rarely modified concurrently (unnecessary complexity)
- Retry indefinitely - set a maximum retry count

## Examples
Please use always these examples as reference: [examples.md](examples.md)
