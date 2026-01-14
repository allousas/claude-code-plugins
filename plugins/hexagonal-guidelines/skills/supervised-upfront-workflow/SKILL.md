---
name: supervised-upfront-workflow
description: Supervised upfront design workflow for hexagonal architecture. Design all layers upfront in plan mode, get human approval, then implement with tests. Single approval checkpoint.
user-invocable: true
---

# Supervised Upfront Workflow

A skill that guides upfront design for hexagonal architecture with human approval before implementation.

It uses Claude's built-in plan mode to design the complete architecture first, then presents it for approval before coding.

## What This Provides

This skill combines:
1. **Upfront Design** - Complete architecture design before coding
2. **Supervised Workflow** - Human approval checkpoint for entire design
3. **Structured Implementation** - Implement all layers after approval

## When to Use

- Want complete architecture review before coding
- Prefer single approval checkpoint over iterative TDD
- Need holistic hexagonal design across all layers
- Complex features benefit from seeing full picture upfront

---

## Flow


### Check Compliance with Guidelines

- Check shallowly if existing codebase follows hexagonal-implementation-guidelines skill.
    - If yes, proceed.
    - If no, check if it follows hexagonal architecture in general:
        - If yes, proceed but adapt to existing patterns, inform the user.
        - If no, alert user about non-compliance and abort.

### Phase 1: Enter Plan Mode & Design

Use Claude's built-in plan mode (or manual design process) to design complete hexagonal architecture for the requested feature:

1. **Explore Codebase**
   - Understand existing patterns and conventions
   - Find similar features for reference
   - Identify integration points

2. **Design All Layers**:
   - **Domain** (aggregates, entities, VOs, events, business rules)
   - **Application** (services, ports, orchestration)
   - **Inbound adapters** (controllers, API contracts)
   - **Outbound adapters** (repositories, external APIs, publishers)

3. **Load Guidelines**
   - Reference hexagonal-ddd-patterns as needed
   - Follow project conventions
   - Consider testing strategy

**For refactors/fixes:** start from the affected layer and continue as needed.

### Phase 2: Present Design for Approval

**Present structured summary using AskUserQuestion tool:**

```
I've designed a complete hexagonal architecture for [feature].

What I've Found:
- Existing patterns: [patterns discovered with file:line]
- Similar features: [reference implementations]
- Integration points: [services, databases, events]

The Proposed Design:

### Domain Layer
**Aggregates:**
- [AggregateName] - [purpose]
  - Entities: [entity names]
  - Value Objects: [VO names with validation rules]
  - Invariants: [key business rules]

**Domain Events:**
- [EventName] - Triggered when [condition]

**Business Rules:**
- [Rule 1 with reasoning]
- [Rule 2 with reasoning]

### Application Layer
**Use Cases (Command Services):**
- [ServiceName] - [purpose]
  - Input: [request structure]
  - Output: [response structure]
  - Event published: [event name]

**Queries (if applicable):**
- [QueryHandlerName] - [purpose]

**Outbound Ports:**
- [PortName] - [why needed]

**Transaction Boundaries:**
- [Where @Transactional applied with reasoning]

### Inbound Adapters
**Type:** [REST/Messaging/Scheduler]

**API Contracts (if REST):**
- POST /api/[resource] - [purpose]
  - Request: [structure]
  - Response: 201 with [structure]
  - Errors: 400, 409, etc.

**Message Contracts (if messaging):**
- Topic: [topic name]
- Event: [event structure]

### Outbound Adapters
**Repositories:**
- [RepositoryName] - [persistence strategy]
  - Database: [schema details]
  - Key fields: [primary keys, indexes]

**External Services:**
- [ServiceName] - [API called, why]

**Event Publishers:**
- [Publisher type] - [events published]

### Testing Strategy
- **Domain/Application:** Unit tests (>90% coverage)
- **Adapters:** Integration tests with [TestContainers/mocks]
- **End-to-end:** Component tests for [key flows]

My Recommendation:
This design follows:
1. Existing patterns from [fileA.kt:line, fileB.kt:line]
2. Project conventions for [specific aspects]
3. Hexagonal architecture principles

Trade-offs considered:
- [Decision 1]: Chose [X] over [Y] because [reasoning]
- [Decision 2]: Chose [X] over [Y] because [reasoning]

Does this design work for you? Any changes needed?
```

**CRITICAL:**
- Show complete design across all layers
- Reference existing patterns with file:line
- Explain key architectural decisions
- Present trade-offs considered
- Always include your recommendation

### Phase 3: CHECKPOINT - Design Approval

**Wait for user response:**

**If approved:**
- Proceed to implementation

**If changes requested:**
- Adjust design based on feedback
- Re-present updated design
- Wait for approval

**If deferred:**
- Document what needs clarification
- Ask specific questions
- Wait for decisions

### Phase 4: Implement with Tests

Once approved:

1. **Implement All Layers** (follow design)
   - Domain layer first (pure business logic)
   - Application layer (use cases, orchestration)
   - Infrastructure adapters (inbound and outbound)
   - Configuration

2. **Add Comprehensive Tests**
   - Domain: Unit tests (>90% coverage)
   - Application: Unit tests (>90% coverage)
   - Adapters: Integration tests
   - End-to-end: Component tests

3. **Follow TDD Within Layers**
   - Write test → RED
   - Implement → GREEN
   - Refactor → Keep GREEN

### Phase 5: Verify

1. **Run All Tests**
   - Execute complete test suite
   - All tests must be GREEN
   - Fix failures immediately

2. **Verify Coverage**
   - Check domain/application >90%
   - Ensure critical paths tested

3. **Clean Up**
   - Remove unused code
   - Clean imports
   - Apply linters

### Phase 6: Validation

**Architecture Compliance Review:**

**Load Reference Materials:**
- hexagonal-implementation-guidelines
- Project conventions
- Team guidelines

**Check Against:**
1. Hexagonal Architecture compliance
2. DDD Principles adherence
3. Code Quality standards

**If violations found:**
- List all violations (file:line, severity)
- Suggest fixes
- Ask whether to fix now or defer

### Phase 7: Report Completion

```
🎉 IMPLEMENTATION COMPLETE

📦 Deliverables:
✅ Implemented (as designed):
  - Domain: [aggregates, VOs, events]
  - Application: [services, ports]
  - Inbound: [controllers/consumers with APIs]
  - Outbound: [repositories/clients]

✅ Tests:
  - Domain coverage: [X%]
  - Application coverage: [Y%]
  - Integration tests: [Z tests]
  - All passing: ✅

✅ Quality Validation:
  - Architecture: COMPLIANT ✅
  - Code Quality: PASSED ✅
  - Conventions: FOLLOWED ✅

🧠 Design Decisions Implemented:
1. [Decision]: [How it turned out]
2. [Decision]: [How it turned out]

📊 Build Status:
  - Tests: [X passed, Y total] ✅
  - Build: SUCCESS ✅

Implementation matches approved design! 🚀
```

---

## Supervision Principles

### Single Approval Checkpoint
- Design everything upfront
- Get approval once for complete design
- Implement without further interruption (unless issues found)

### Comprehensive Design
- Show complete picture across all layers
- Include API contracts, database schemas, events
- Explain architectural decisions with trade-offs
- Reference existing patterns

### Structured Communication
- Use consistent design presentation format
- Provide context from codebase exploration
- Explain trade-offs clearly
- Always include recommendation

### Flexibility
- Allow design iterations based on feedback
- Don't proceed until approval received
- Adjust based on user's architectural preferences

---

## When to Use This vs TDD Workflow

**Use Supervised Upfront Workflow When:**
- Complex features benefit from seeing full design
- Team prefers architecture review before coding
- Want single approval point instead of multiple checkpoints
- Design decisions span multiple layers

**Use Supervised TDD Workflow When:**
- Prefer iterative development with frequent checkpoints
- Want to validate each layer before moving to next
- Requirements emerge during implementation
- Simpler features with clear scope

---

## Mandatory Rules

### Design Completeness
- Design all layers before seeking approval
- Include testing strategy
- Reference existing patterns
- Explain key decisions

### Quality Standards
- **>90% coverage**: Domain and application layers
- **Architecture compliance**: Validate at end
- **Fix violations**: Address when found

### Implementation Fidelity
- Implement according to approved design
- If changes needed during implementation, ask user
- Don't deviate from approved architecture without discussion
