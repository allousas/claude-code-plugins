---
name: aligning-existing-code-with-guidelines
description: Apply when applying (or planning) changes and existing codebase patterns conflict or differ from any loaded claude skill code patterns. Always ask the user whether to follow the skill recommendations or maintain consistency with existing code.
---

## Purpose
Loaded skills provide best practice guidelines, but existing codebases often use different patterns. This skill ensures you always ask the user which approach to follow rather than imposing guidelines that break codebase consistency.

## When to Use
When you detect conflicts between skill guidelines and existing code patterns

## Process

### 1. Identify the Conflict
When you notice the existing code contradicts a loaded skill guideline, identify:
- What the skill guideline recommends
- What the existing codebase actually does
- Specific examples with file:line references

### 2. Always Ask the User
- **Never blindly apply skill guidelines or current code style.** Present both options and Ask the user which approach to follow

### 3. Respect the Decision
After the user chooses, apply that approach consistently throughout your implementation. Don't question or second-guess their choice.
