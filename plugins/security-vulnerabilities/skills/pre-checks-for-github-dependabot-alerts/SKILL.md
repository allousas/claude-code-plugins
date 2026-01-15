---
name: pre-checks-for-github-dependabot-alerts
description: Performs prerequisites check and extracts GitHub account/repo from local git remote for Dependabot operations. Use when working with Dependabot alerts.
---

# Pre-checks for GitHub Dependabot Alerts

This skill performs common pre-checks required for all Dependabot alert operations:
1. Verifies the `$GITHUB_DEPENDABOT_PAT` environment variable is set
2. Extracts GitHub account and repository name from local git remote

## Pre-checks: Step Check Required Env Vars

Check if the `$GITHUB_DEPENDABOT_PAT` environment variable is set without printing its value:

```bash
if [ -z "$GITHUB_DEPENDABOT_PAT" ]; then
  echo "❌ GITHUB_DEPENDABOT_PAT environment variable is not set"
  echo "Please set this environment variable with a valid GitHub Personal Access Token that has 'security_events' scope"
  exit 1
fi
echo "✅ GITHUB_DEPENDABOT_PAT is configured"
```

**If the variable is empty or not set:**
- Stop execution immediately
- Inform the user they need to set this environment variable with a valid GitHub Personal Access Token that has `security_events` scope

## Pre-checks: Step Extract GitHub Repository Info

Extract the GitHub account and repository name from the git remote URL:

```bash
git remote get-url origin
```

Parse the output to extract the account and repo name. The URL will typically be in one of these formats:
- HTTPS: `https://github.com/ACCOUNT/REPO.git`
- SSH: `git@github.com:ACCOUNT/REPO.git`

**Parsing logic:**
- For HTTPS URLs: Extract from `github.com/` to `.git`
- For SSH URLs: Extract from `:` to `.git`
- Split by `/` to get ACCOUNT and REPO
- Remove `.git` suffix if present

**If the command fails or the URL is not a GitHub URL:**
- Stop execution with error message: "Could not extract GitHub repository information from git remote. Please ensure you're in a git repository with a GitHub remote."

## Pre-checks: Step Return Values

Store and return the following values for use by the calling command:
- `ACCOUNT`: GitHub account/organization name
- `REPO`: Repository name

Display to the user:
```
✓ Prerequisites check passed
✓ Repository: ACCOUNT/REPO
```

These values will be available for subsequent operations that require GitHub API calls.
