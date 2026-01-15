---
argument-hint: [alert-id] [skip-tests=false] [human-in-the-loop=true]
description: Fix a specific Dependabot security alert and verify all tests pass
---


# Fix Dependabot Alert

- Fetches detailed information about the specified alert
- Checks if the vulnerability is already fixed
- If already fixed: Dismisses the alert with proof (asks first if human-in-the-loop mode)
- If not fixed:
    - Updates the vulnerable dependency
    - Runs all tests to ensure nothing breaks
    - Verifies the vulnerable library is removed
    - Creates a commit with the security fix
- Handles breaking changes (always asks for approval first)

## Step 0: Identify Parameters

IMPORTANT: Before starting, state the actual values received:
- **ALERT_ID** = $1
- **SKIP_TESTS** = $2 (if empty/not provided, treat as "false")
- **HUMAN_IN_LOOP** = $3 (if empty/not provided, treat as "true")

Execution mode:
- SKIP_TESTS="false" → Run all tests
- SKIP_TESTS="true" → Only build
- HUMAN_IN_LOOP="false" → Automatic mode, NO user questions except breaking changes
- HUMAN_IN_LOOP="true" → Interactive mode, ask for confirmations

## Step 1: Pre-checks

IMPORTANT: Load the pre-checks skill 'pre-checks-for-github-dependabot-alerts' skill

**If pre-checks fail, stop execution.**

## Step 2: Fetch Alert Details

Retrieve the specific alert details from GitHub and put it in context:

```bash
curl -H "Authorization: Bearer $GITHUB_DEPENDABOT_PAT" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/[ACCOUNT]/[REPO]/dependabot/alerts/$1
```

**If the alert does not exist (404 or other error), inform the user and stop execution.**

Show the alert details to the user.

## Step 3: Fix the Issue

**First, check if the alert is already fixed:** Verify whether the vulnerable dependency has already been updated or removed from the project. Use appropriate dependency commands (e.g., `gradle dependencies`, `npm list`, `pip show`, etc.) to check the current state.

**If the alert is already fixed:**
- Show the proof to the user (include command output demonstrating the dependency is no longer vulnerable)
- Check HUMAN_IN_LOOP value:
  - If HUMAN_IN_LOOP="false": Proceed directly to dismiss (do NOT ask)
  - If HUMAN_IN_LOOP="true": Ask "Do you want to dismiss this alert?"
- After confirmation (or in automatic mode), dismiss the alert via GitHub API with state "dismissed" and reason "not_used"
- Include proof in the dismissal comment showing the dependency is no longer vulnerable
- Use the following API call, ensure to use dismissed_reason with value not_used:
  ```bash
  curl -X PATCH \
    -H "Authorization: Bearer $GITHUB_DEPENDABOT_PAT" \
    -H "Accept: application/vnd.github+json" \
    https://api.github.com/repos/[ACCOUNT]/[REPO]/dependabot/alerts/$1 \
    -d '{"state":"dismissed","dismissed_reason":"not_used","dismissed_comment":"Alert already fixed. Proof: [include verification output here]"}'
  ```
- Skip to Step 5 (Summary) and report the dismissal

**If the alert is not yet fixed:**
- Fix the security vulnerability.
- Run validation based on SKIP_TESTS:
  - If SKIP_TESTS="false": Run all tests (they must pass)
  - If SKIP_TESTS="true": Only run build (must succeed)
- If tests/build fail, check HUMAN_IN_LOOP:
  - If HUMAN_IN_LOOP="false": Report failure and stop (do NOT ask)
  - If HUMAN_IN_LOOP="true": Ask user what to do
- **If the vulnerability involves a library/dependency fix, ensure the vulnerable library is completely removed from the final artifact.** Verify using appropriate dependency commands and collect proof for the summary report.
- **If the fix requires a breaking change (major version update, API changes, etc.):** Always ask the user to proceed before applying the fix, regardless of human-in-the-loop mode
- **Proof that vulnerability has been fixed** 

## Step 4: Create Commit

**IMPORTANT**: Prepend any info needed to the comment, taking in account commit history 

Use the following commit message template :
- **If the alert was fixed with a dependency update:**
  ```
  fix: address Dependabot alert #$1
  Resolves: https://github.com/[ACCOUNT]/[REPO]/security/dependabot/$1
  Co-Authored-By: Claude <noreply@anthropic.com>
  ```
- **If the alert was dismissed (already fixed):**
  ```
  chore: dismiss Dependabot alert #$1 - already fixed
  Alert was already resolved. Dismissed via API with proof.
  https://github.com/[ACCOUNT]/[REPO]/security/dependabot/$1
  Co-Authored-By: Claude <noreply@anthropic.com>
  ```

Check HUMAN_IN_LOOP value:
- If HUMAN_IN_LOOP="false": Use template and create commit immediately (do NOT ask)
- If HUMAN_IN_LOOP="true": Show template and ask user to confirm or modify

Create the commit with all modified files, if no changes, commit without body.

## Step 5: Summary

Report to the user:
- Alert ID fixed
- Package and versions (before/after)
- Vulnerability details (CVE, severity)
- Test results
- Files modified
- Commit created

ARGUMENTS: $1 (Alert ID) $2 (Skip tests, default: false) $3 (Human-in-the-loop, default: true)
