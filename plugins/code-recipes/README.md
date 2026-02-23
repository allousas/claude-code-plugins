# Code Recipes

**A set of Claude Code plugins organized by language, providing implementation patterns and refactoring recipes.**

**IMPORTANT-NOTE:** As mentioned in the main readme, this is experimental, to learn claude capabilities and not intended to be used in prod environments.

## Structure

Code Recipes is organized as a collection of plugins grouped by language and concern:

```
code-recipes/
  kotlin/
    building-blocks/       # Implementation patterns for architectural components
    refactoring/           # Recipes for aligning and improving existing code
```

Each subfolder contains skills that Claude Code loads contextually when relevant to your current task.

## Kotlin

### Building Blocks

Implementation patterns for common architectural components.

- **Implementing Application Services** - Guidelines and examples for creating application services that orchestrate infrastructure and domain to execute business use cases. Covers naming, dependency injection, transaction management, and keeping business logic in the domain layer.

### Refactoring

Recipes for improving and aligning existing code with guidelines.

- **Aligning Existing Code with Guidelines** - When loaded skills conflict with existing codebase patterns, this recipe ensures Claude always asks you which approach to follow rather than silently breaking codebase consistency.