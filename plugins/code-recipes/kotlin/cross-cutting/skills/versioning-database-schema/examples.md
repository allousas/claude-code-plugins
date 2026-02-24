# Examples

## Initial table creation
```sql
-- V001__create_team_table.sql
CREATE TABLE IF NOT EXISTS team (
    id         UUID PRIMARY KEY,
    name       VARCHAR(255) NOT NULL,
    version    INTEGER      NOT NULL DEFAULT 0,
    created_at TIMESTAMP    NOT NULL DEFAULT now(),
    updated_at TIMESTAMP    NOT NULL DEFAULT now()
);
```

## Table with JSONB column
```sql
-- V002__create_annual_leave_table.sql
CREATE TABLE IF NOT EXISTS annual_leave (
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
-- V004__add_country_code_to_team.sql
ALTER TABLE team ADD COLUMN country_code VARCHAR(3);
```

## Adding an index
```sql
-- V005__add_index_on_team_name.sql
CREATE INDEX IF NOT EXISTS idx_team_name ON team (name);
```

## Non-blocking index creation (PostgreSQL)
```sql
-- V006__add_index_on_annual_leave_employee.sql
-- CONCURRENTLY avoids locking the table during index creation
-- Flyway must run this migration outside a transaction
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_annual_leave_employee_id ON annual_leave (employee_id);
```

## Flyway Spring Boot configuration
```yaml
# application.yaml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
```