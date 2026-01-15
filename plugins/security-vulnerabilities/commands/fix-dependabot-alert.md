---
argument-hint: [alert-id] [skip-tests=false]
description: Fix a specific Dependabot security alert and verify all tests pass
---


# Fix Dependabot Alert

- Fetches detailed information about the specified alert
- Checks if the vulnerability is already fixed
- If already fixed: Offers to dismiss the alert with proof
- If not fixed:
    - Updates the vulnerable dependency
    - Runs all tests to ensure nothing breaks
    - Verifies the vulnerable library is removed
    - Creates a commit with the security fix
- Handles breaking changes by asking for approval first

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
- Ask the user if they want to dismiss the alert
- If user confirms, dismiss the alert via GitHub API with state "dismissed" and reason "not_used" as appropriate
- Include proof in the dismissal comment showing the dependency is no longer vulnerable
- Use the following API call:
  ```bash
  curl -X PATCH \
    -H "Authorization: Bearer $GITHUB_DEPENDABOT_PAT" \
    -H "Accept: application/vnd.github+json" \
    https://api.github.com/repos/[ACCOUNT]/[REPO]/dependabot/alerts/$1 \
    -d '{"state":"dismissed","dismissed_reason":"not_used","dismissed_comment":"Alert already fixed. Proof: [include verification output here]"}'
  ```
- Skip to Step 4 & 5 (Summary) and report the dismissal

**If the alert is not yet fixed:**
- Fix the security vulnerability. 
- Proceed with:
  - If {$2} is false **All tests must run and pass**
  - Else **Just check that project builds successfully**
  - Ask user what to do if fails
- **If the vulnerability involves a library/dependency fix, ensure the vulnerable library is completely removed from the final artifact.** Verify using appropriate dependency commands and collect proof for the summary report.
- **If the fix requires a breaking change (major version update, API changes, etc.), ask the user to proceed before applying the fix.**
- **Proof that vulnerability has been fixed** 

## Step 4: Create Commit

**Always ask the user for the commit message before creating the commit.**

When asking, suggest including the alert link in the commit message:
`https://github.com/[ACCOUNT]/[REPO]/security/dependabot/$1`

After receiving the commit message, create the commit with all modified files.

## Step 5: Summary

Report to the user:
- Alert ID fixed
- Package and versions (before/after)
- Vulnerability details (CVE, severity)
- Test results
- Files modified
- Commit created

ARGUMENTS: $1 (Alert ID)
