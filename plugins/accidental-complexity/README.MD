# Accidental Complexity

**Claude Code plugin that identifies and analyzes accidental complexity in your codebase through automated analysis.**

**IMPORTANT NOTE:** As mentioned in the root readme, this is experimental and not intended to be used in prod environments. This plugin is designed for learning and exploration of claude patterns.

## The Problem

Accidental complexity accumulates silently in codebases:
- Unnecessary abstractions and indirection make code harder to understand
- Framework and library tax adds cognitive overhead without business value
- Over-engineering creates maintenance burden

## The Solution

This plugin provides agents and commands to systematically identify accidental complexity patterns in your codebase. It generates structured findings and actionable reports to help you simplify and improve code quality.

**Why not a single prompt?** 

[Context rot](https://research.trychroma.com/context-rot): In really long prompts, attention scores distribute across more tokens, diluting the influence of system agent instructions. 


## Available Commands

### /analyse

Orchestrates the complete accidental complexity analysis workflow.

```bash
/accidental-complexity:analyse "all classes in the service layer" 
```

**Output files:**
- `accidental-complexity-file-queue.json` - Queue tracking file analysis status (pending/analyzed)
- `accidental-complexity-findings.jsonl` - Raw findings (one JSON object per violation)
- `accidental-complexity-report-[YYYY-MM-DD].md` - Final report with summary of the findings

## Accidental complexity patterns detection list

Check [here](./skills/accidental-complexity-patterns-list/SKILL.md) the actual used list to detect at file level

