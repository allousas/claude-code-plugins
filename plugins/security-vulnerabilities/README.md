# Security Vulnerabilities

**Claude Code plugin that helps to fix Dependabot security dependabot alerts in your github repositories.**

IMPORTANT-NOTE: As mentioned in the main readme, this is experimental, to learn claude capabilities and not intended to be used in prod environments.
IMPORTANT-NOTE 2: Results are not yet optimal, agent tends to hallucinate and produce unexpected results or omit parameters, please use it for learning purposes.

## The Problem

Security vulnerabilities pile up:
- Dependabot creates alerts but doesn't fix them
- Manual fixes are time-consuming and repetitive
- Tests need to be run after each fix
- Dismissing false positives requires API calls and verification

## The Solution

This plugin automates the entire security fix workflow:

1. **Fetches Dependabot alerts** - Gets vulnerability details from GitHub API
2. **Analyzes the fix** - Understands the vulnerability and required upgrade
3. **Applies the fix** - Updates dependencies and verifies removal
4. **Runs tests** - Ensures nothing breaks
5. **Creates commit** - Commits the fix with proper documentation
6. **Dismisses false positives** - Auto-dismisses already-fixed alerts with proof

## Quick Start

### Prerequisites

1. **Navigate to your GitHub repository:**
   ```bash
   cd /path/to/your/github/repo
   ```

   The plugin automatically extracts the GitHub account and repository name from your local git remote configuration.

2. **Set up your GitHub Personal Access Token:**
   ```bash
   export GITHUB_DEPENDABOT_PAT="your-github-token"
   ```

   **Required token scopes:**
   - `dependabot-*` - Read and write Dependabot alerts

### Available Commands

#### 1. List Dependabot Alerts

View all open security alerts in your repository:

```bash
/security-vulnerabilities:list-dependabot-alerts
```

See [list-dependabot-alerts command](commands/list-dependabot-alerts.md) for details.

**Use this command when:**
- You want to see what security vulnerabilities exist
- You need to prioritize which alerts to fix first
- You want to get alert IDs for targeted fixes

#### 2. Fix Dependabot Alert

Fix a specific security alert:

```bash
/security-vulnerabilities:fix-dependabot-alert 123
```

See [fix-dependabot-alert command](commands/fix-dependabot-alert.md) for details.

**Use this command when:**
- You want to fix a specific vulnerability
- You need fine-grained control over which alerts to address
- You want to fix high-priority alerts first

#### 3. Fix All Dependabot Alerts

Automatically fix all open security alerts:

```bash
/security-vulnerabilities:fix-all-dependabot-alerts
```

See [fix-all-dependabot-alerts command](commands/fix-all-dependabot-alerts.md) for details.

**Use this command when:**
- You want to fix all vulnerabilities at once
- You're doing a security audit cleanup
- You have multiple alerts and want batch processing

### Example Workflow

```bash
# 1. Navigate to your repository
cd ~/projects/my-app

# 2. List all alerts to see what needs fixing
/security-vulnerabilities:list-dependabot-alerts

# 3. Option A: Fix a specific critical alert first
/security-vulnerabilities:fix-dependabot-alert 45

# 3. Option B: Fix all alerts at once
/security-vulnerabilities:fix-all-dependabot-alerts
```