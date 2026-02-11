---
name: accidental-complexity-patterns-list
description: List of common accidental complexity patterns to look for in codebases
---

# 🎯 What is Accidental Complexity?

Self-inflicted complexity from tools, abstractions, or design choices that make code harder than it needs to be.

These patterns are optimized for detection on a single file, to be used by the single-file-accidental-complexity-analyzer subagent, but can also be used as a checklist for manual code reviews or refactoring sessions.

# Accidental Complexity – Patterns List

## 1. Mixed Responsibilities

- Execution flow within a single class where ≥ 3 distinct concerns (domain logic, error handling, mapping, control flow, I/O) are interleaved rather than separated into named steps or collaborators.

## 2. Destructive Decoupling

- Modules (Gradle / Maven) are introduced but change, release, and deploy together
- Modules exist only to mirror layers or packages, used solely to apply architectural constraints
- Events/observers/callbacks replace direct calls with:
    - 0 async execution
    - 0 fan-out (>1 consumer)
    - ≥ 1 business decision ( hiding business steps )

## 3. Framework & Lib Tax

- Method signature includes ≥ 1 lib wrapper type (suspend, Either, Result, Flow, Mono, Flux, etc.) AND:
    - the method body contains 0 abstraction-specific operations
    - the abstraction is returned or forwarded unchanged
- Function signature includes async/coroutine modifier (suspend, async, etc.) but function body contains zero async operations: no await expressions, no async suspension primitives, no async context switches, and no async collection/stream terminal operations
- Method signatures expose library-error abstractions with generic error types (Error, DomainError, AppError, Failure) that add no information beyond exceptions.
- Expressions where ≥3 chained calls and ≥70% of method invocations are library-specific, obscuring domain logic.
- ORM or data-access framework usage when simple queries (≤1 join) require ≥2 framework-specific classes or generated code.
- Data-access code where simple queries (≤1 join) are expressed without explicit SQL, relying solely on ORM abstractions or generated queries.

## 4. Unnecessary Indirection

- Flag any dependency where one service/use-case class injects, calls, or otherwise depends on another service/use-case class.
- Deep chains of private method calls (>=2) fragment required business steps and obscure execution flow
- Pass-through methods with 1 forwarded call, 0 branching, and no value transformation.

## 5. Clever Code

- Expressions where lambda nesting depth ≥ 3.
- Flag chained scope-function or lambda calls with chain length ≥ 3 in a single expression
- Regex used with no groups, alternation, or lookarounds where simple string operations suffice
- Bitwise or arithmetic tricks without explanatory naming or comments.

## 6. Naming & Structural Noise

- Redundant generic class suffixes (Impl, Default, Base, Utils), allow specific (Repository, Controller, Service, Constants ...)
- Flag names where ≥50% of tokens are implementation terms (UserManager, DataHandler) outside the infrastructure layers.
- When a type's name communicates a responsibility that contradicts its placement in the architecture (e.g. transformation utilities living in service or domain layers)

## 7. Side effects

- Catch blocks that log an exception and rethrow it while adding 0 new context.
- Flag execution paths within same class ≥3 logging statements and ≤1 decision point.
- Logging statements placed immediately before or after every step.
- Domain model methods with state mutation ≥1, side effects ≥1, and non-void return type.

# Pattern Names (use these exact strings)
- "mixed-responsibilities"
- "destructive-decoupling"
- "framework-lib-tax"
- "unnecessary-indirection"
- "clever-code"
- "naming-noise"
- "side-effects"