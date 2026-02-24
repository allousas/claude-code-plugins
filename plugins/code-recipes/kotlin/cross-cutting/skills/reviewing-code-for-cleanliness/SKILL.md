---
name: reviewing-code-for-cleanliness
description: Apply only when reviewing any Kotlin code for readability and maintainability.
---

## Purpose
Write Kotlin code that is readable, explicit, and maintainable. Favor clarity over cleverness, keeping functions small and intentions obvious.

## Guidelines

Besides the already known clean code guidelines, here are some additional recommendations:

**DO:**
- Name classes, functions, and variables to reveal intent - the name should make the code self-documenting
- Keep functions short and focused on a single responsibility
- Use early returns to reduce nesting depth
- Use `when` expressions for multi-branch logic instead of if-else chains
- Use data classes for value objects and DTOs
- Use extension functions to add behavior to existing types without inheritance
- Use named arguments for functions with multiple parameters of the same type
- Prefer immutable data (`val` over `var`, immutable collections)

**DON'T:**
- Call private functions from other private functions — keep the flow centralized in the public method
- Chain Kotlin scope functions (`let`, `also`, `run`, `apply`, `with`) - one level is fine, chaining obscures flow
- Use regex for simple string manipulation - use built-in string functions instead
- Use single-letter variable names except in lambdas with obvious context (e.g., `{ it.name }`)
- Create deeply nested code (more than 2 levels of indentation inside a function or lambda body)
- Use `it` in multi-line lambdas - name the parameter explicitly
- Add comments that restate what the code does - restructure the code to be self-explanatory instead
- Use nullable types when a value is always present - reserve nullability for genuinely optional data
- Create utility classes with static methods - use top-level functions or extension functions instead
