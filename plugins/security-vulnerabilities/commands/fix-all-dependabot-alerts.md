---
argument-hint: none
description: Fix all open Dependabot security alerts one by one
---

# Fix All Dependabot Alerts

This command will:
1. Fetch all open Dependabot alerts
2. Fix each alert one by one by calling the fix-dependabot-alert command
3. Track progress and report results

## Step 1: Pre-checks

IMPORTANT: Load the pre-checks skill 'pre-checks-for-github-dependabot-alerts' skill

**If pre-checks fail, stop execution.**

## Step 2: Fetch All Alerts

Retrieve all open Dependabot alerts from GitHub and put them in context:

```bash
curl -H "Authorization: Bearer $GITHUB_DEPENDABOT_PAT" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/[ACCOUNT]/[REPO]/dependabot/alerts?state=open
```

**If there are no alerts or an error occurs, inform the user and stop execution.**

Show the user:
- Total number of open alerts found
- List of alert IDs with package names and severity levels

## Step 3: Fix Alerts One by One

For each alert found, call the fix-dependabot-alert command:

```
/security-vulnerabilities:fix-dependabot-alert [alert-id]
```

Process alerts sequentially (one at a time, not in parallel).

After each alert is processed:
- Report success or failure for that specific alert
- Continue to the next alert even if the current one failed

## Step 4: Final Summary

Report to the user:
- Total alerts processed
- Number of alerts successfully fixed
- Number of alerts that failed to fix
- List of alert IDs and their status (fixed/failed)

ARGUMENTS: None (automatically extracts from local git remote)
