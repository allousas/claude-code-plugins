---
name: setting-up-a-new-project
description: Apply when bootstrapping a new microservice project from scratch (e.g. Kotlin/Spring Boot with Gradle, or TypeScript/NestJS with pnpm). Always confirm the language, default tech stack and architecture with the user before generating any code.
---

## Purpose
Bootstrap a new microservice project with the correct structure, dependencies, configuration, and conventions.

## Before You Start — Confirm Defaults

**CRITICAL: Before generating any code, present the defaults below and ask the user to confirm or override.**

1. Determine the language/stack (from the user's request or existing repo context); if unclear, ask
2. Present that stack's default tech stack table from its examples file, plus the default architecture
3. Ask: _"I'll set up the project with these defaults. OK, or do you want to change anything?"_

### Default Tech Stack
Defined per language in the examples. Every stack covers the same categories: language/runtime, framework, database (PostgreSQL), DB access, migrations, build tool and package manager, JSON, HTTP clients, test framework, mocking, assertions, containers for integration tests, HTTP stubs, API testing, linting.

### Default Architecture
Hexagonal (DDD style): domain, application, infrastructure (inbound, outbound, config) — see `architecture:applying-pragmatic-hexagonal`.

### Required Input
Only ask the user for:
- **Service name** (e.g. `team-service`)
- The language-specific project identifier listed in the examples (e.g. base package, package name)

Everything else uses defaults unless the user says otherwise.

## Project Generation 

After confirmation, generate the project:
- Build/package configuration with pinned dependency versions and the standard scripts/tasks (build, run, test, integration test, lint, database migrate)
- Architecture folders with a health endpoint as the only feature
- Local database via Docker Compose and a first (empty or example) migration
- Unit and integration test setup, integration tests starting their own database
- Verify the project builds, lints and tests pass before handing over

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
