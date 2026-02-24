# Claude Code Plugins Collection

A collection of Claude Code plugins to enhance development workflows with AI-powered automation.

**NOTE:** Experimental — learning and exploring Claude Code capabilities. Not intended for production use.

## Installation

```shell
/plugin marketplace add allousas/claude-code-plugins
/plugin install <plugin-name>
```

## Plugins

### Security Vulnerabilities

Fixes Dependabot security alerts in GitHub repositories.

```bash
export GITHUB_DEPENDABOT_PAT="your-token"  # needs dependabot-* scopes
```

| Command | Description |
|---------|-------------|
| `/security-vulnerabilities:list-dependabot-alerts` | List all open alerts |
| `/security-vulnerabilities:fix-dependabot-alert 123` | Fix a specific alert |
| `/security-vulnerabilities:fix-all-dependabot-alerts` | Fix all open alerts |

### Code Insights

Agents that identify accidental complexity and code quality issues in Kotlin codebases.

**Usage:** navigate to your project and run:
```
use accidental-complexity-analyser subagent to analyse all kotlin prod files in this project, please skip test files and configuration.
```

**Output:**

| File | Description |
|------|-------------|
| `accidental-complexity-findings.jsonl` | Line-by-line findings with pattern IDs |
| `accidental-complexity-report-[YYYY-MM-DD].md` | Summary with top patterns and statistics |

### Code Recipes — Kotlin

Set of claude code skills recipes for Kotlin microservices, organized by concern. Skills are auto-loaded by Claude when relevant to the current task.

#### Architecture (`kotlin-architecture`)

| Skill | Description | Triggered when |
|-------|-------------|----------------|
| applying-pragmatic-hexagonal | Hexagonal architecture with pragmatic shortcuts | Designing or reviewing project structure |
| applying-pragmatic-layered | Layered architecture style | Designing or reviewing project structure |

#### Bootstrapping (`kotlin-bootstrapping`)

| Skill | Description | Triggered when |
|-------|-------------|----------------|
| setting-up-a-new-project | Bootstrap with default tech stack and hexagonal structure | Creating a new microservice from scratch |

#### Building Blocks (`kotlin-building-blocks`)

| Skill | Description | Triggered when |
|-------|-------------|----------------|
| implementing-application-services | Application services that orchestrate business use cases | Creating or changing application services |
| implementing-controllers | HTTP controllers (inbound adapters) | Creating or changing REST controllers |
| implementing-kafka-consumers | Kafka message consumers | Creating or changing Kafka consumers |
| implementing-kafka-producers | Kafka message producers | Creating or changing Kafka producers |
| implementing-repositories | Database repositories (outbound adapters) | Creating or changing repositories |
| implementing-service-integrations | External service integrations (outbound HTTP) | Creating or changing HTTP clients |

#### Cross-Cutting (`kotlin-cross-cutting`)

| Skill | Description | Triggered when |
|-------|-------------|----------------|
| configuring-runtime-dependencies | Spring configuration and dependency wiring | Setting up Spring beans and config |
| handling-errors-with-either | Functional error handling with Arrow's Either | Implementing error handling with Either |
| handling-errors-with-exceptions | Exception-based error handling | Implementing error handling with exceptions |
| reviewing-code-for-cleanliness | Code cleanliness review checklist | Reviewing code quality |
| versioning-database-schema | Flyway database migrations | Creating or changing database schema |
| writing-tests | Testing guidelines and patterns | Writing unit, integration, or component tests |

#### Patterns (`kotlin-patterns`)

| Skill | Description | Triggered when |
|-------|-------------|----------------|
| applying-domain-event-publisher | Domain event publishing (in-memory, Kafka-backed) | Implementing domain events |
| applying-optimistic-locking | Optimistic locking for concurrent access | Implementing concurrency control |
| applying-outbox-pattern | Transactional outbox for reliable event publishing | Implementing reliable messaging |

#### Refactoring (`kotlin-refactoring`)

| Skill | Description | Triggered when |
|-------|-------------|----------------|
| aligning-existing-code-with-guidelines | Asks before overriding existing codebase patterns | Loaded skills conflict with existing code style |

## Customizations

Fork the repo and make your changes. See [testing locally](https://code.claude.com/docs/en/plugins#test-your-plugins-locally) and [distributing](https://code.claude.com/docs/en/plugin-marketplaces).
