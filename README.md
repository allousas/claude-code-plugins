# Claude Code Plugins Collection

A collection of Claude Code plugins to enhance development workflows with AI-powered automation.

**IMPORTANT NOTE**:This repository is my way of learning and experimenting with Claude Code capabilities. 
It is not intended for production use. Please review and test thoroughly before using in real projects. 
Result are not optimal yet, agents may hallucinate or produce unexpected results. Use for learning and exploration purposes.

## Available Plugins

### 1. Hexagonal Guidelines
Maintains consistent hexagonal architecture patterns across microservices ecosystems.

**Key Features:**
- Hexagonal implementation guidelines with customizable references
- Guided implementation workflows with human checkpoints
- Architecture auditor for compliance checking
- Migration planner for existing code

[View Documentation →](./plugins/hexagonal-guidelines/README.md)

### 2. Security Vulnerabilities
Automatically fixes Dependabot security alerts in your repositories.

**Key Features:**
- Fetches and analyzes Dependabot alerts
- Applies fixes and runs tests
- Creates commits with proper documentation
- Auto-dismisses false positives

[View Documentation →](./plugins/security-vulnerabilities/README.md)

### 3. Code Insights
Set of agents that analyze codebases to provide different insights, starting with accidental complexity analysis.

**Key Features:**
- Detects 20+ complexity anti-patterns across 7 categories
- Analyzes cognitive overload, destructive decoupling, and framework tax
- Structured JSONL findings with file:line references
- Comprehensive markdown reports with pattern statistics

[View Documentation →](./plugins/code-insights/README.md)

## Installation

### Option 1: Install from GitHub (Recommended)

Add to your Claude Code marketplace this repo:

```shell
/plugin marketplace add allousas/claude-code-plugins
```

Then install the plugins:

```bash
# Install hexagonal guidelines
/plugin install hexagonal-guidelines

# Install security vulnerabilities
/plugin install security-vulnerabilities

# Install code insights
/plugin install code-insights
```

### Customizations

Please fork the repository and make your changes. 

Then follow: 
- testing locally: https://code.claude.com/docs/en/plugins#test-your-plugins-locally
- Distributing: https://code.claude.com/docs/en/plugin-marketplaces
