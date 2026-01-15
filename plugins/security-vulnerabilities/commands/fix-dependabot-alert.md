---
argument-hint: [alert-id] [skip-tests=false] [human-in-the-loop=true]
description: Fix a specific Dependabot security alert and verify all tests pass
---

# Fix Dependabot Alert

Fetches alert details, fixes or dismisses if already resolved, runs tests, creates commit.

## Parameters

State received values:
- **ALERT_ID** = $1
- **SKIP_TESTS** = $2 (default: false) → false=run tests, true=build only
- **HUMAN_IN_LOOP** = $3 (default: true) → false=automatic mode, true=ask user for confirmations

**Note**: Breaking changes always require approval regardless of mode.

## Step 1: Pre-checks

Load 'pre-checks-for-github-dependabot-alerts' skill. Stop if pre-checks fail.

## Step 2: Fetch Alert

Fetch alert via API and show to user:
```bash
curl -H "Authorization: Bearer $GITHUB_DEPENDABOT_PAT" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/[ACCOUNT]/[REPO]/dependabot/alerts/$1
```
Stop if alert doesn't exist (404).

## Step 3: Check & Fix

Verify vulnerable dependency using appropriate commands (`gradle dependencies`, `npm list`, `pip show`, etc.). 
**Important:** Check test dependencies too.

**If already fixed:**
- Show proof (command output)
- Ask to dismiss if HUMAN_IN_LOOP=true, otherwise proceed directly
- Dismiss via API with reason "not_used" and proof in comment:
  ```bash
  curl -X PATCH -H "Authorization: Bearer $GITHUB_DEPENDABOT_PAT" \
    -H "Accept: application/vnd.github+json" \
    https://api.github.com/repos/[ACCOUNT]/[REPO]/dependabot/alerts/$1 \
    -d '{"state":"dismissed","dismissed_reason":"not_used","dismissed_comment":"Alert already fixed. Proof: [verification output]"}'
  ```

**If not fixed:**
- Fix the vulnerability
- Validate: Run tests (SKIP_TESTS=false) or build only (SKIP_TESTS=true)
  - If validation fails try to fix it
  - If validation still fail: Report and stop (HUMAN_IN_LOOP = false) or ask user (HUMAN_IN_LOOP = true)
- Verify vulnerable library removed from artifact, collect proof
- **Breaking changes:** Always ask for approval first (regardless HUMAN_IN_LOOP value) 

## Step 4: Create Commit

Commit message templates:
```
# Fixed alert:
fix: address Dependabot alert #$1
Resolves: https://github.com/[ACCOUNT]/[REPO]/security/dependabot/$1
Co-Authored-By: Claude <noreply@anthropic.com>

# Dismissed alert:
chore: dismiss Dependabot alert #$1 - already fixed
Alert was already resolved. Dismissed via API with proof.
https://github.com/[ACCOUNT]/[REPO]/security/dependabot/$1
Co-Authored-By: Claude <noreply@anthropic.com>
```

**Prepend `[{TASK-ID}]` if branch name contains a task id.**

Ask user to confirm (HUMAN_IN_LOOP = true) or commit immediately (HUMAN_IN_LOOP = false).

## Step 5: Summary

Report to the user:
- Alert ID fixed
- Package and versions (before/after)
- Vulnerability details (CVE, severity)
- Test results
- Files modified
- Commit created

ARGUMENTS: $1 (Alert ID) $2 (Skip tests, default: false) $3 (Human-in-the-loop, default: true)
