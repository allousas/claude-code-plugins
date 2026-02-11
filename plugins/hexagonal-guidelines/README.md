# Hexagonal Guidelines

**Claude Code plugin that maintains consistent hexagonal architecture guidelines and patterns across big microservices ecosystems.**

**IMPORTANT NOTE:** As mentioned in the root readme, this is experimental and not intended to be used in prod environments, 
for now this plugin does **not generate the expected results** and I personally consider that it **is over-engineered**. 

## The Problem

Your microservices drift architecturally over time:
- Service A constructs value objects in aggregates
- Service B constructs them in application services
- Service C mixes both approaches

This happens because architectural decisions live in people's heads, not in accessible documentation.

## The Solution

This plugin provides:

1. **Hexagonal implementation guidelines** - Your team's/company hexagonal patterns documented in markdown with code examples, with customizable references.
2. **Guided Implementation** - Two workflows (TDD layer-by-layer, or upfront design) with **Human Checkpoints** - Claude asks for approval at key decision points
3. **Architecture Auditor** - Automated agent that audits your codebase for violations
4. **Migration planner** - To be done

When Claude implements features, it follows your documented patterns consistently.

## Quick Start

### 1. Use it

Start a project from scratch and just write:

```bash
# TDD: Layer-by-layer with checkpoints after each layer
/hexagonal-guidelines:implement-supervised "Allow users to set a numeric code when closing a locker"

# Upfront: Design everything, approve once, implement
/hexagonal-guidelines:implement-supervised supervised-upfront-workflow "Code should be encrypted"

# Check architecture compliance 
Use the architecture-auditor sub-agent to analyze this codebase
```

You can also do it in existent codebases, but it will follow the guidelines, you can disable the 
skill `hexagonal-implementation-guidelines` or adapt it to your/company style

### 2. Fork and Customize with your own team or company specific hexagonal guidelines
Fork this repo and edit `hexagonal-guidelines/skills/hexagonal-implementation-guidelines/references/`:
- `layers/domain-layer.md`, `layers/app-services-layer.md`, `layers/infra-inbound.md`, `layers/infra-outbound.md`, `layers/infra-config.md`
- `code-conventions.md`, `error-handling.md`, `test-guidelines.md`, `tech-stack.md`, `hexagonal-shortcuts.md`, `side-effects.md`

For testing distribution check [this](../../README.md#customizations) 

**Critical:** Use code from YOUR actual services, not generic examples. Real code makes Claude follow your specific patterns.


---

## Example: How It Prevents Drift

Your rule: "Value objects must be constructed inside aggregates."

Without this plugin:
- Service A: Creates VOs in aggregates ✅
- Service B: Creates VOs in app services ❌
- Service C: Mix of both ❌

With this plugin, you document it once in `layers/domain-layer.md`:

```kotlin
// ✅ CORRECT
data class Locker(val lockerId: UUID, val code: NumericCode?) {
    fun close(numericCode: String): Locker {
        val code = NumericCode.create(numericCode)  // VO created here
        return this.copy(code = code, status = CLOSED)
    }
}

// ❌ WRONG
@Service
class CloseLockerService {
    operator fun invoke(lockerId: UUID, code: String) {
        val numericCode = NumericCode.create(code)  // Don't do this
        // ...
    }
}
```

Now all services follow the same pattern.

---

## When to Use This

**Use it when:**
- Ecosystem with 5+ microservices with hexagonal architecture
- Architectural drift is causing problems, different projects and teams with different patterns
- Patterns are established and stable
- You value structured workflows with checkpoints

**Skip it when:**
- Single service (use `.claude/CLAUDE.md` instead)
- Architecture is still experimental
- Team maintains consistency naturally

---
