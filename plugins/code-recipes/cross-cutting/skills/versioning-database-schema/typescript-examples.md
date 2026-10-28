# Examples (dbmate)

## Naming and location
```
db/migrations/
├── 20260910120000_create_teams_table.sql
├── 20260911093000_create_annual_leaves_table.sql
├── 20260912101500_create_outbox_table.sql
└── 20260915140000_add_country_code_to_teams.sql
```
- `<YYYYMMDDHHMMSS>` — UTC timestamp prefix, sorts in apply order
- `_` — single underscore separator
- `<description>` — snake_case summary of the change
- Generate new files with `pnpm db:new <description>` (`dbmate new`); each file has a `-- migrate:up` section and an empty `-- migrate:down` marker (forward-only)

## Initial table creation
```sql
-- 20260910120000_create_teams_table.sql
-- migrate:up
CREATE TABLE IF NOT EXISTS teams (
    id         UUID PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    version    INTEGER      NOT NULL DEFAULT 0,
    created_at TIMESTAMP    NOT NULL DEFAULT now(),
    updated_at TIMESTAMP    NOT NULL DEFAULT now()
);

-- migrate:down
```

## Table with JSONB column
```sql
-- 20260911093000_create_annual_leaves_table.sql
-- migrate:up
CREATE TABLE IF NOT EXISTS annual_leaves (
    id          UUID PRIMARY KEY,
    employee_id UUID         NOT NULL,
    year        INTEGER      NOT NULL,
    leaves      JSONB        NOT NULL DEFAULT '[]'::jsonb,
    version     INTEGER      NOT NULL DEFAULT 0,
    created_at  TIMESTAMP    NOT NULL DEFAULT now(),
    updated_at  TIMESTAMP    NOT NULL DEFAULT now(),
    UNIQUE (employee_id, year)
);

-- migrate:down
```

## Outbox table for reliable event publishing
```sql
-- 20260912101500_create_outbox_table.sql
-- migrate:up
CREATE TABLE IF NOT EXISTS outbox (
    id           UUID PRIMARY KEY,
    aggregate_id UUID         NOT NULL,
    event_type   VARCHAR(255) NOT NULL,
    payload      JSONB        NOT NULL,
    published    BOOLEAN      NOT NULL DEFAULT false,
    created_at   TIMESTAMP    NOT NULL DEFAULT now()
);

-- Partial index: only scan unpublished entries
CREATE INDEX idx_outbox_unpublished ON outbox (created_at) WHERE published = false;

-- migrate:down
```

## Adding a column
```sql
-- 20260915140000_add_country_code_to_teams.sql
-- migrate:up
ALTER TABLE teams ADD COLUMN country_code VARCHAR(3);

-- migrate:down
```

## Adding an index
```sql
-- 20260916090000_add_index_on_teams_name.sql
-- migrate:up
CREATE INDEX IF NOT EXISTS idx_teams_name ON teams (name);

-- migrate:down
```

## Non-blocking index creation (PostgreSQL)
```sql
-- 20260917110000_add_index_on_annual_leaves_employee.sql
-- migrate:up transaction:false
-- CONCURRENTLY avoids locking the table during index creation
-- dbmate must run this migration outside a transaction (transaction:false above)
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_annual_leaves_employee_id ON annual_leaves (employee_id);

-- migrate:down
```

## dbmate configuration
`package.json` (excerpt):
```json
{
  "scripts": {
    "db:migrate": "dbmate --migrations-dir ./db/migrations --no-dump-schema up",
    "db:status": "dbmate --migrations-dir ./db/migrations status",
    "db:new": "dbmate --migrations-dir ./db/migrations new"
  }
}
```
```bash
# .env — read by dbmate
DATABASE_URL=postgres://mydomain:mydomain@localhost:5432/mydomain?sslmode=disable
```

## TypeScript / dbmate specifics
- dbmate is a standalone binary (not an npm dependency) that reads `DATABASE_URL` (from the environment or `.env`)
- Migrations are not applied on application startup — run `pnpm db:migrate` as a deploy step (or init container) before the new version starts
- dbmate records applied migrations in `schema_migrations` but does not checksum them — editing an applied file is silently ignored, so the immutability rule relies on review
- Don't fill `-- migrate:down` sections; fix forward with a new migration
- Migrations are plain SQL; no TypeScript migration code. Keep the Kysely `Database` schema types (`infrastructure/outbound/database.schema.ts`) in sync with every migration in the same change
- Use `-- migrate:up transaction:false` for statements that can't run in a transaction (e.g. `CREATE INDEX CONCURRENTLY`)
- For local Postgres containers without TLS, append `?sslmode=disable` to `DATABASE_URL`
- Integration tests apply migrations by invoking the real `dbmate` binary against the Testcontainers database via the same migration files (see `writing-tests`)
