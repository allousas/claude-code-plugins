# Claude Code Plugins Collection

A collection of Claude Code plugins to enhance your development workflow with AI-powered automation.

**IMPORTANT NOTE: This repository is my way of learning and experimenting with Claude Code capabilities. It is not intended for production use. Please review and test thoroughly before using in real projects.**

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

### 3. Architecture Insights
Identifies and assesses architectural debt based on customizable constraints.

**Key Features:**
- Customizable constraint definition (max 10 constraints)
- Automated codebase scanning for violations
- Prioritized assessment with severity scores
- Actionable recommendations grouped by constraint

[View Documentation →](./plugins/architecture-insights/README.md)

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

# Install architecture insights
/plugin install architecture-insights
```

### Customizations

Please fork the repository and make your changes. 

Then follow: 
- testing locally: https://code.claude.com/docs/en/plugins#test-your-plugins-locally
- Distributing: https://code.claude.com/docs/en/plugin-marketplaces
