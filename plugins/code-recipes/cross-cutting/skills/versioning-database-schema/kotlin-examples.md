# Examples (Flyway)

## Naming and location
```
src/main/resources/db/migration/
├── V001__create_teams_table.sql
├── V002__create_annual_leaves_table.sql
├── V003__create_outbox_table.sql
└── V004__add_country_code_to_teams.sql
```
- `V` prefix for versioned migrations (executed once, in order)
- `<NNN>` — zero-padded sequential number: `V001`, `V002`, ..., `V042`
- `__` — double underscore separator
- `<description>` — snake_case summary of the change

## Initial table creation
```sql
-- V001__create_teams_table.sql
CREATE TABLE IF NOT EXISTS teams (
    id         UUID PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    version    INTEGER      NOT NULL DEFAULT 0,
    created_at TIMESTAMP    NOT NULL DEFAULT now(),
    updated_at TIMESTAMP    NOT NULL DEFAULT now()
);
```

## Table with JSONB column
```sql
-- V002__create_annual_leaves_table.sql
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
```

## Outbox table for reliable event publishing
```sql
-- V003__create_outbox_table.sql
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
```

## Adding a column
```sql
-- V004__add_country_code_to_teams.sql
ALTER TABLE teams ADD COLUMN country_code VARCHAR(3);
```

## Adding an index
```sql
-- V005__add_index_on_teams_name.sql
CREATE INDEX IF NOT EXISTS idx_teams_name ON teams (name);
```

## Non-blocking index creation (PostgreSQL)
```sql
-- V006__add_index_on_annual_leaves_employee.sql
-- CONCURRENTLY avoids locking the table during index creation
-- Flyway must run this migration outside a transaction
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_annual_leaves_employee_id ON annual_leaves (employee_id);
```

## Flyway Spring Boot configuration
```yaml
# application.yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
```

## Spring / Flyway specifics
- Spring Boot auto-configures Flyway when `org.flywaydb:flyway-core` is on the classpath
- Migrations run automatically on application startup from `src/main/resources/db/migration/`
- Use `spring.flyway.*` properties for configuration (url, user, password, locations)
- Flyway validates checksums of applied migrations — editing one fails startup
- Don't write `U` (undo) migrations; fix forward with a new `V` migration
- In tests, Flyway runs against the Testcontainers database via the same migration files
