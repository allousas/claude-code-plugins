# Security Vulnerabilities

**Claude Code plugin that automatically fixes Dependabot security alerts in your repositories.**

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

**What it does:**
- Fetches all open Dependabot alerts from GitHub
- Displays a summary table with:
  - Alert ID
  - Severity level (critical/high/medium/low)
  - Package name and vulnerable version
  - Brief vulnerability description
  - CVE ID (if available)
- Sorts alerts by severity for easier prioritization

**Use this command when:**
- You want to see what security vulnerabilities exist
- You need to prioritize which alerts to fix first
- You want to get alert IDs for targeted fixes

#### 2. Fix Dependabot Alert

Fix a specific security alert:

```bash
/security-vulnerabilities:fix-dependabot-alert 123
```

**What it does:**
- Fetches detailed information about the specified alert
- Checks if the vulnerability is already fixed
- If already fixed: Offers to dismiss the alert with proof
- If not fixed:
  - Updates the vulnerable dependency
  - Runs all tests to ensure nothing breaks
  - Verifies the vulnerable library is removed
  - Creates a commit with the security fix
- Handles breaking changes by asking for approval first

**Use this command when:**
- You want to fix a specific vulnerability
- You need fine-grained control over which alerts to address
- You want to fix high-priority alerts first

#### 3. Fix All Dependabot Alerts

Automatically fix all open security alerts:

```bash
/security-vulnerabilities:fix-all-dependabot-alerts
```

**What it does:**
- Fetches all open Dependabot alerts
- Processes each alert sequentially (one at a time)
- For each alert, calls the fix-dependabot-alert command
- Continues even if some alerts fail to fix
- Provides a final summary with success/failure counts

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