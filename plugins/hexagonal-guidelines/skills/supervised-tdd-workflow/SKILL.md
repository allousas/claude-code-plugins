---
name: supervised-tdd-workflow
description: Supervised TDD workflow for hexagonal architecture. Guides outside-in implementation with human-in-the-loop checkpoints at each layer. Combines structured supervision with test-driven development.
user-invocable: true
---

# Supervised TDD Workflow

A skill that guides test-driven development for hexagonal architecture with human supervision at key decision points.

## What This Provides

This skill combines:
1. **TDD Methodology** - Outside-in, layer-by-layer implementation (Inbound → Application → Domain → Outbound)
2. **Supervised Workflow** - Human checkpoints for decisions and progress reviews
3. **Structured Communication** - Consistent question format with recommendations

## Core Workflow

### Analysis Phase

1. **Understand Requirements**
   - Parse request completely
   - Explore codebase for context

2. **Determine Flow Type**
   - **New Feature Flow** → Implement all layers
   - **Fix/Refactor Flow** → Start from affected layer

### Initial Setup

**Track Progress** (use TodoWrite for multi-layer implementations, skip for simple changes)

Suggested TODO structure:
```
- Implement Inbound layer
- Implement Application layer
- Implement Domain layer
- Implement Outbound layer
- End-to-end verification
- Architecture validation
```

### Check Compliance with Guidelines

- Check shallowly if existing codebase follows hexagonal-implementation-guidelines skill.
  - If yes, proceed.
  - If no, check if it follows hexagonal architecture in general:
    - If yes, proceed but adapt to existing patterns, inform the user.
    - If no, alert user about non-compliance and abort.

---

## Layer Implementation (Outside-In TDD)

### Order of Layer Implementation (IMPORTANT)

**Follow always:** Inbound → Application → Domain → Outbound

**For refactors/fixes:** start from the affected layer and continue as needed.

### For Each Layer:

#### CHECKPOINT 1: Before Layer Implementation (Decision Point)

**Ask about layer-specific details if ambiguous or not inferable:**

| Layer | Common Questions |
|-------|-----------------|
| **Inbound** | Trigger type (HTTP/messaging/scheduler)? API contracts? Error responses? Status codes? |
| **Application** | Outbound ports needed? Domain events to publish? Transaction boundaries? |
| **Domain** | Aggregate root? Business rules? Value objects vs primitives? Domain event definitions? |
| **Outbound** | Persistence strategy? Database schema? External APIs? Event publishing mechanism? |

**How to Ask (Required Format):**

Use **AskUserQuestion tool**:

```
[One sentence describing the decision]

What I've Found:
- [Codebase exploration results with file:line]
- [Existing patterns discovered]
- [Relevant conventions]

The Decision:
[Specific question]

Options:

Option A: [Approach]
✅ Pros:
  - [Benefit 1]
  - [Benefit 2]
❌ Cons:
  - [Drawback 1]

Option B: [Approach]
✅ Pros:
  - [Benefit 1]
❌ Cons:
  - [Drawback 1]

My Recommendation:
I recommend Option [A/B] because:
1. [Primary reason based on patterns - fileA.kt:line]
2. [Secondary reason based on principles]
3. [How it aligns with architecture]

Does this approach work for you?
```

**CRITICAL:** Never ask without including your recommendation.

**When NOT to Ask (Infer Instead):**
- Existing patterns (API endpoint paths, HTTP status codes, error formats)
- Requirements clearly stated (CRUD operations, REST conventions)
- Conventions documented (framework patterns, DI, architecture rules)
- Best practices (security standards, validation, testing fundamentals)

#### Implementation: TDD Red-Green-Refactor Cycle

**Follow strictly:**
1. Write test (integration for adapters, unit for app/domain)
2. Run test → Must be RED
3. Write minimal code to compile
4. Implement logic per loaded guidelines
5. Run test → Must be GREEN
6. REFACTOR (improve design, keep green)
7. REPEAT until all test cases covered
8. Review adjacent layers - update if needed (keep all tests green)
9. Run all tests → Must be GREEN

**Never proceed to next layer with failing tests.**

#### CHECKPOINT 2: After Layer Implementation (Review Point)

**Show layer completion summary:**

```
✅ Checkpoint: [Layer] Complete

Implementation Summary:
📝 Files Modified/Created:
  - src/.../ClassName.kt:123 - [description]
  - test/.../TestClass.kt:45 - [test scenarios]

🧪 Tests:
  - [X tests passed, Y total]
  - Happy path: ✅
  - Error cases: ✅
  - Edge cases: ✅

[If applicable:]
🔌 APIs: POST /api/resource, GET /api/resource/:id
📢 Events: ResourceCreated, ResourceUpdated
🔗 Integrations: [External service calls]

Next: [What comes next]

Ready to continue?
```

Wait for user approval before proceeding to next layer.

#### Next Layer

Repeat CHECKPOINT 1 → Implementation → CHECKPOINT 2 for next layer in sequence.

---

## Verification Phase

### End-to-End Tests

- Cover full end-to-end flows (Component tests)
- Test happy paths
- Test critical error scenarios (only if really needed)
- Execute complete test suite
- All tests must be GREEN

### Cleanup Phase

1. **Remove Unused Code**
   - Delete unused classes/methods
   - Remove dead code
   - Remove empty folders
   - Clean up imports
   - Apply linters if defined

2. **Verify Tests Still Pass**
   - Run full test suite
   - Must remain GREEN

---

## Validation Phase

### Architecture Compliance Review

**Load Reference Materials:**
- hexagonal and ddd skills if present
- Project conventions
- Team guidelines

**Check Against (priority order):**
1. Project guidelines
2. Specific skills (hexagonal-implementation-guidelines)
3. Company standards
4. General knowledge

**Areas to validate:**
1. Hexagonal Architecture compliance
2. DDD Principles adherence
3. Code Quality standards

### CHECKPOINT 3: Violations Found (If Any)

**If violations found:**
- List all violations (file:line, severity)
- Suggest fixes
- Ask: "Should violations be fixed now or deferred?"

Wait for user decision on how to proceed.

---

## Final Completion Report

```
🎉 IMPLEMENTATION COMPLETE

📦 Deliverables:
✅ Implemented:
  - [Component 1] at file.kt:line
  - [Component 2] at file.kt:line

✅ Tests:
  - Coverage: [X/Y tests]
  - All passing: ✅

✅ Quality Validation:
  - Architecture: COMPLIANT ✅
  - Code Quality: PASSED ✅
  - Conventions: FOLLOWED ✅

🧠 Key Decisions Made:
1. [Decision]: [Reasoning with evidence]
2. [Decision]: [Reasoning with evidence]

📊 Build Status:
  - Tests: [X passed, Y total] ✅
  - Build: SUCCESS ✅

Ready for use! 🚀
```

---

## Supervision Principles

### Always Recommend
- Show what you've researched with file:line references
- Present 2-4 concrete options with trade-offs
- **Always include your recommendation** with clear reasoning
- Base recommendations on patterns/conventions/research

### Progressive Disclosure
- Ask layer-specific questions at layer time
- Let details emerge naturally during implementation
- Avoid premature architectural decisions

### Structured Communication
- Use consistent question format (shown above)
- Provide context from codebase exploration
- Explain trade-offs clearly
- Report progress after each checkpoint

---

## Mandatory Rules

### TDD Discipline
- **Always test first**: No code without tests
- **Red-Green-Refactor**: Follow strictly
- **Fix before proceeding**: Never move forward with RED tests

### Quality Standards
- **>90% coverage**: Every class/method tested
- **Architecture compliance**: Validate at end
- **Fix violations**: Address when found

### Progress Tracking Guidelines

Use TodoWrite based on complexity:

**When to use:**
- New features spanning 2+ layers
- Complex refactors affecting multiple files
- Multi-step fixes requiring coordination

**When to skip:**
- Simple bug fixes in one layer
- Single endpoint additions
- Minor refactors confined to one file
