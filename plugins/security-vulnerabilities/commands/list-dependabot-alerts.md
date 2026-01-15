---
argument-hint: none
description: List all open Dependabot security alerts with IDs, severity, and brief summary
---

# List Dependabot Alerts

This command will:
1. Fetch all open Dependabot alerts
2. Display a summary with alert IDs, severity levels, and brief descriptions

## Step 1: Pre-checks

IMPORTANT: Load the pre-checks skill 'pre-checks-for-github-dependabot-alerts' skill

**If pre-checks fail, stop execution.**

## Step 2: Fetch All Alerts

Retrieve all open Dependabot alerts from GitHub:

```bash
curl -H "Authorization: Bearer $GITHUB_DEPENDABOT_PAT" \
  -H "Accept: application/vnd.github+json" \
  "https://api.github.com/repos/[ACCOUNT]/[REPO]/dependabot/alerts?state=open"
```


**If there are no alerts or an error occurs, inform the user and stop execution.**

## Step 3: Display Summary

Present the alerts to the user in a clean, readable format, Sorted by severity :

For each alert, display:
**Alert Summary:**

- **Alert ID:** [id]
- **Summary:** [brief description of the vulnerability]
- **URL:** [link to the alert on GitHub]

Finally:
**Total Alerts:** [{use jq 'length', on the prev response, if not working, just count by yourself}]

## Step 4: Next Steps

Inform the user they can:
- Fix a specific alert: `/security-vulnerabilities:fix-dependabot-alert [ALERT-ID]`
- Fix all alerts: `/security-vulnerabilities:fix-all-dependabot-alerts`

Display the extracted ACCOUNT and REPO values for easy reference.

ARGUMENTS: None (automatically extracts from local git remote)