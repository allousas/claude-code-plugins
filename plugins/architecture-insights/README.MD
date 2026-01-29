# Architecture Insights

**Claude Code plugin that identifies and assesses architectural debt based on customizable constraints.**

**IMPORTANT NOTE:** As mentioned in the root readme, this is experimental and not intended to be used in prod environments. This plugin is designed for learning and exploration of architectural patterns and technical debt.

## The Problem

Technical debt accumulates silently:
- Architectural violations aren't caught until code review
- No systematic way to identify patterns of architectural drift
- Teams lack visibility into where architecture principles are broken
- Debt assessment is inconsistent and subjective

## The Solution

This plugin provides:

1. **Customizable Constraint Definition** - Define your project's architectural constraints in an editable checklist (max 10)
2. **Automated Debt Discovery** - Scans codebase in batches to identify violations
3. **Prioritized Assessment** - Generates reports with severity, effort, and reversibility scores
4. **Actionable Recommendations** - Groups findings by constraint with clear next steps

## Available Agents

### architectural-debt-assessor

Scans your codebase against customizable architectural constraints and generates a prioritized technical debt assessment report.

```bash
# Assess architectural debt
Use the architectural-debt-assessor agent to assess this codebase
```

The agent will create an editable checklist (max 10 constraints), scan your code for violations, and generate detailed reports with severity scores and recommendations.

