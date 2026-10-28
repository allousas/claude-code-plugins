---
name: reviewing-code-for-cleanliness
description: ALWAYS invoke this skill when, and ONLY when, the user explicitly asks to review or audit code for readability, maintainability or clean code. Covers naming, nesting, function size, comments, immutability and idiomatic language constructs. Do not use it for regular implementation work.
---

## Purpose
Write code that is readable, explicit, and maintainable. Favor clarity over cleverness, keeping functions small and intentions obvious.

## Guidelines

Besides the already known clean code guidelines, here are some additional recommendations:

**DO:**
- Name classes, functions, and variables to reveal intent - the name should make the code self-documenting
- Keep functions short and focused on a single responsibility — under ~30 lines whenever possible
- Use early returns to reduce nesting depth
- Use exhaustive match/switch expressions for multi-branch logic instead of if-else chains
- Use immutable value types for value objects and DTOs
- Name parameters at the call site (named arguments, or a parameter object) for functions with multiple parameters of the same type
- Prefer immutable data (read-only bindings, immutable collections); create new objects instead of mutating existing ones
- Name implementations by what distinguishes them (`PostgresTeamRepository`), never with an `Impl` suffix
- Keep comments for essential "why" context or non-obvious workarounds only

**DON'T:**
- Call private functions from other private functions — keep the flow centralized in the public method
- Chain fluent/functional helpers in ways that obscure control flow - one level is fine
- Use regex for simple string manipulation - use built-in string functions instead
- Use single-letter variable names except in short lambdas with obvious context (e.g., `{ it.name }`, `(t) => t.name`)
- Create deeply nested code (more than 2 levels of indentation inside a function or lambda body)
- Leave multi-line lambdas with implicit/unnamed parameters - name the parameter explicitly
- Add comments that restate what the code does - restructure the code to be self-explanatory instead
- Use nullable/optional types when a value is always present - reserve them for genuinely optional data
- Create utility classes with only static methods - use top-level/module functions (or extensions where the language has them) instead

## Examples
Always use the examples for the project's language as reference:
- Kotlin: [kotlin-examples.md](kotlin-examples.md)
- TypeScript: [typescript-examples.md](typescript-examples.md)
