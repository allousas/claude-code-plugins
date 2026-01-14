# Side Effects Guidelines

## Core Principle: Side Effects at the Boundaries

Side effects should be performed at the boundaries of the application (inbound/outbound adapters), not in the core domain or application services.

## Rules

**Where to perform side effects:**
- **Inbound adapters** (controllers, consumers, schedulers) - HTTP responses, logging requests
- **Outbound adapters** (repositories, HTTP clients, message producers) - Database writes, external API calls, message publishing
- **Domain event subscribers** - Logging, metrics, publishing to message brokers

**Where NOT to perform side effects:**
- **Domain layer** - Keep pure, no side effects (no I/O, no logging, no state mutation outside the aggregate)
- **Application services** - Avoid direct side effects, use domain events pattern instead

**Domain Events Pattern:**
- Use domain events to trigger side effects in response to state changes
- Application services publish domain events
- Domain event subscribers handle side effects (logging, metrics, message publishing)
- Keeps core logic pure and side effects isolated

**Return Types:**
- Functions that perform side effects should return `Unit` (no meaningful return value)
- Example: `fun sendEmail(...)` returns `Unit`