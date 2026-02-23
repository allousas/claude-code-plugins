# Code Insights

**Set of agents Claude Code plugin that ...**

**IMPORTANT-NOTE:** As mentioned in the main readme, this is experimental, to learn claude capabilities and not intended to be used in prod environments.

## Accidental Complexity Analyser Agent

### Usage

Run claude and invoke the agent: 
```
use accidental-complexity-analyser subagent to analyse all kotlin prod files in this project, please skip test files and configuration.
```

#### The Problem

**Accidental complexity** is complexity that comes from abstractions, design choices or frameworks, that make the code harder to understand and maintain.

Accidental complexity accumulates over time:
- Unnecessary abstractions make code harder to understand
- Framework overhead obscures business logic
- Indirection layers fragment execution flow
- Poor naming and clever code reduce maintainability

#### The Solution

This plugin analyzes Kotlin files to detect complexity patterns:

1. **Scans Kotlin files** - Finds all production source files
2. **Analyzes patterns** - Detects 20+ complexity anti-patterns across 7 categories
3. **Generates findings** - Creates structured JSONL with file:line references
4. **Produces report** - Summarizes patterns by category with affected files

### Output

The analyzer generates two files:

- `accidental-complexity-findings.jsonl` - Line-by-line findings with pattern IDs
- `accidental-complexity-report-[YYYY-MM-DD].md` - Summary with top patterns and statistics
