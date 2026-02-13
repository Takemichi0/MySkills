---
name: end-branch
description: Complete branch work by creating PR to main, merging after checks pass, and cleaning up
user-invocable: true
model: sonnet
allowed-tools: Bash(git *), Bash(gh pr create *), Bash(gh pr checks *), Bash(gh pr view *), Bash(gh api *), Bash(docker *)
---

# Branch Completion and Cleanup

Complete your work on a feature branch created with `/start-branch` by verifying clean state, creating a pull request to merge into main, waiting for CI/CD checks, merging, and cleaning up the branch.

This skill assumes all commits and pushes have been completed using `/end-session` in previous work sessions.

## Workflow

### 1. Verify Current State and Clean Working Tree

1. **Verify git repository**:
   - Check if in a git repository: `git rev-parse --is-inside-work-tree`
   - If not in a git repository, inform the user and STOP execution

2. **Get current branch**:
   - Get current branch name: `git branch --show-current`
   - Verify we're NOT on main or master branch
   - If on main/master, inform user this skill is for feature branches only and STOP execution

3. **Verify clean state**:
   - Check for uncommitted changes: `git status --porcelain`
   - If there are uncommitted changes:
     - Display the uncommitted files
     - Inform user: "There are uncommitted changes. Please commit and push them using `/end-session` first."
     - STOP execution
   - If working tree is clean: proceed to next step

4. **Verify pushed to remote**:
   - Check if current branch has remote tracking: `git rev-parse --abbrev-ref @{u}`
   - Compare local and remote commits:
     - Local: `git rev-parse HEAD`
     - Remote: `git rev-parse @{u}`
   - If they differ:
     - Inform user: "Local branch is not in sync with remote. Please push your changes first."
     - STOP execution
   - If in sync: proceed to next step

### 2. Review TODO.md and Verify Task Completion

1. **Analyze commit history**:
   - Get all commits in this branch: `git log origin/main...HEAD --oneline`
   - Get full diff: `git diff origin/main...HEAD`
   - Analyze the commit messages and changes to understand what work was completed in this branch

2. **Check TODO.md**:
   - Check if TODO.md exists in the project root
   - If TODO.md doesn't exist: skip to next step (Create Pull Request)

3. **Review TODO.md against completed work**:
   - Read TODO.md content
   - Identify all tasks that relate to the work completed in this branch
   - Check if all related tasks are marked as completed (checked off)

4. **Handle unchecked tasks**:
   - If all related tasks are already checked: proceed to next step
   - If there are unchecked tasks that should be completed:
     - Display the unchecked tasks to the user
     - Use AskUserQuestion: "Found unchecked tasks in TODO.md that appear to be completed in this branch. Should I mark them as completed?"
       - Options: "Yes, update TODO.md" or "No, leave as is"
     - If user says "Yes, update TODO.md":
       - Update TODO.md to mark the tasks as completed
       - Note: Don't commit yet, will commit together with CLAUDE.md updates if needed
     - If user says "No, leave as is": proceed to next step

5. **Check and Update CLAUDE.md and Other Documentation**:
   1. **Analyze if documentation updates are needed**:
      - Review the completed work and changes made in this branch
      - Determine if there were:
        - Major requirement changes
        - New feature implementations
        - Architectural changes
        - Other significant modifications that should be documented

   2. **If updates are needed**:
      - Use AskUserQuestion: "The work included significant changes. Should I update CLAUDE.md and other documentation files to reflect these changes?"
        - Options: "Yes, update documentation" or "No, skip documentation updates"
      - If user says "No, skip documentation updates":
        - If TODO.md was updated: commit and push TODO.md changes only
        - Then skip to next step (Create Pull Request)

   3. **If user approves documentation updates**:
      - Find all CLAUDE.md files in the project hierarchy
      - Start with the most specific (deepest) CLAUDE.md file
      - Update it to reflect the changes made
      - If the changes are significant, propagate updates to parent CLAUDE.md files up to the project root
      - Check and update other relevant .md files as needed
      - Ensure consistency between CLAUDE.md and TODO.md

6. **Commit and push documentation updates**:
   - If TODO.md or CLAUDE.md or other .md files were updated:
     - Stage all changes: `git add TODO.md CLAUDE.md` (and any other updated .md files)
     - Commit with message using HEREDOC:
       ```bash
       git commit -m "$(cat <<'EOF'
       update documentation with completed work

       Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
       EOF
       )"
       ```
     - Push the changes: `git push`
     - Verify push was successful

### 3. Create Pull Request

1. **Analyze commit history for PR**:
   - Get all commits in this branch: `git log origin/main...HEAD`
   - Get full diff: `git diff origin/main...HEAD`
   - Analyze the changes to understand what was implemented

2. **Generate PR title and body**:
   - Title: Short summary (under 70 characters) based on commit messages and changes
   - Body format:
     ```markdown
     ## Summary
     - <bullet point 1>
     - <bullet point 2>
     - <bullet point 3>

     ## Test plan
     - [ ] <test step 1>
     - [ ] <test step 2>

     Generated with Claude Code
     ```

3. **Create PR to main**:
   ```bash
   gh pr create --base main --title "<title>" --body "$(cat <<'EOF'
   <body content>
   EOF
   )"
   ```
   - Capture PR URL and PR number from output
   - Display PR URL to user

### 4. Monitor CI/CD Checks

1. **Get PR number**: Extract from PR URL or use `gh pr view --json number`

2. **Fetch and analyze Gemini review** (if available):
   - Wait 10-15 seconds for Gemini bot to post review
   - Fetch comments: `gh api repos/:owner/:repo/pulls/<pr-number>/comments --jq '.[] | "\(.user.login): \(.body)"'`
   - Fetch reviews: `gh api repos/:owner/:repo/pulls/<pr-number>/reviews --jq '.[] | "\(.user.login) (\(.state)): \(.body)"'`
   - Filter for Gemini bot (username contains "gemini")
   - For each Gemini suggestion:
     - Read mentioned files to understand context
     - Verify suggestion validity (check against project dependencies, actual code)
     - Categorize: HIGH (bugs/security), MEDIUM (quality), LOW (style)
     - Determine if suggestion improves code
   - Automatically apply valid fixes:
     - Use Edit tool to apply changes
     - Stage: `git add <modified-files>`
     - Commit: `git commit -m "address Gemini review feedback\n\nCo-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>"`
     - Push: `git push`
     - Display summary with reasoning
   - If no valid suggestions, display why and continue

3. **Monitor CI/CD checks**:
   - Use `gh pr checks` to get current check status
   - Wait for all checks to complete
   - Poll every 10-30 seconds until done
   - Display progress to user

4. **Check results**:
   - If all checks pass: proceed to merge confirmation
   - If any checks fail:
     - Display failed check names and details
     - Show PR URL for investigation
     - Use AskUserQuestion: "CI/CD checks failed. Do you want to ignore the failures and proceed with merge anyway?"
       - Options: "Yes, ignore and merge" or "No, stop and fix issues"
     - If user says "Yes, ignore and merge": proceed to merge confirmation step
     - If user says "No, stop and fix issues":
       - STOP execution
       - Inform user they can fix issues, push changes, and run `/end-branch` again

### 5. Confirm Merge with User

- Display PR summary:
  - PR title and URL
  - Number of commits
  - Files changed
  - Check status (all passed or some failed)
- Use AskUserQuestion: "Ready to merge this PR into main?"
  - Options: "Yes, merge now" or "No, cancel"
- If user says "No, cancel":
  - STOP execution
  - Inform user they can merge manually later using the PR URL

### 6. Merge Pull Request

- Execute: `gh pr merge --merge` (create merge commit)
- Verify merge was successful
- GitHub typically auto-deletes the remote branch after merge (if configured)

### 7. Stop Docker Environment

1. **Verify current location**:
   - Confirm we're still in the working directory
   - `pwd` to check current path

2. **Stop Docker containers for this branch only**:
   - Get project name from directory: `PROJECT_NAME=$(basename "$(pwd)")`
   - `docker compose -p "$PROJECT_NAME" down`
   - This stops and removes only containers associated with this project/directory
   - Other worktrees or branches running their own containers are not affected

3. **Verify cleanup**:
   - `docker compose -p "$PROJECT_NAME" ps` to confirm no containers are running for this project

### 8. Switch to Main Branch

1. **Fetch latest changes**:
   - `git fetch origin`
   - This ensures we have the latest main with our merged changes

2. **Switch to main**:
   - `git checkout main`
   - Verify we're on main: `git branch --show-current`

3. **Update local main**:
   - `git pull origin main`
   - This brings our merged changes to local main

### 9. Clean Up Branch

1. **Delete local branch**:
   - `git branch -d <branch-name>`
   - If it fails with unmerged warning, use `-D` (force delete)
   - The branch should be merged, so `-d` should work

2. **Verify remote branch deletion**:
   - `git ls-remote --heads origin <branch-name>` to confirm remote branch was deleted
   - GitHub automatically deletes the remote branch after PR merge (if auto-delete is enabled)
   - If remote branch still exists: `git push origin --delete <branch-name>` to delete it

3. **Clean up tracking references**:
   - `git fetch --prune` to remove stale remote-tracking branches

### 10. Final Verification and Summary

- Run `git branch -a | grep <branch-name>` to confirm branch is deleted
- Display summary:
  - PR merged successfully into main
  - Docker containers stopped
  - Local and remote branches cleaned up
  - Current branch: main
  - Ready for next task

## Important Notes

- This skill should only be used for feature branches created with `/start-branch`
- All commits must be pushed before running this skill (use `/end-session` to commit and push)
- The skill will verify TODO.md and update it if needed before creating the PR
- If significant changes were made, CLAUDE.md and other documentation files will be updated (with user confirmation)
- Documentation updates are committed and pushed before creating the PR
- The PR always merges into main (the branch was created from main)
- CI/CD checks must pass before merging (or user can choose to ignore failures)
- User confirmation is required before merge
- Docker containers are stopped after merge (assumes Docker environment was started with `/start-branch`)
- Both local and remote branches are deleted after successful merge
- Returns to main branch after cleanup
- The skill will stop at any failure point and inform the user

## Why Merge Commit Instead of Rebase?

This skill uses `--merge` (create merge commit) instead of `--rebase` for the following reasons:

- Reflects actual development history: Merge commits preserve the true timeline of how the work was done
- Same conflict resolution: Both merge and rebase can encounter conflicts
- Preserves context: Merge commits make it easy to see all changes that were part of a feature
- Simpler and safer: Merge commits don't rewrite history
- Team collaboration: Easier to understand for team members

Use rebase only when you have a compelling reason, such as needing to remove sensitive data or fix a critical mistake in commit history.

## Example Flow

```
User runs: /end-branch

[Verify git repository: OK]
[Current branch: feature/add-user-authentication]
[Check git status...]
Working tree clean: OK

[Check remote sync...]
Local HEAD: abc123
Remote HEAD: abc123
In sync: OK

[Review TODO.md...]
[Execute: git log origin/main...HEAD --oneline]
Found 3 commits:
- add login page
- add authentication middleware
- add user session management

[Check TODO.md...]
TODO.md found, checking related tasks...
Found completed work: User authentication feature
All related tasks are already checked: OK

[Check documentation updates...]
Analyzing changes: New authentication feature with middleware
Ask: "The work included significant changes. Should I update CLAUDE.md and other documentation files to reflect these changes?"
User answers: "Yes, update documentation"

[Update CLAUDE.md...]
Found CLAUDE.md at project root
Updating to reflect new authentication feature
CLAUDE.md updated successfully

[Commit and push documentation updates...]
[Execute: git add TODO.md CLAUDE.md]
[Execute: git commit -m "update documentation with completed work"]
[Execute: git push]
Documentation updates pushed successfully

[Analyze commits for PR...]
[Execute: git log origin/main...HEAD]
Found 3 commits to merge

PR title: "Add user authentication feature"

[Execute: gh pr create --base main --title "..." --body "..."]
PR created: https://github.com/user/my-app/pull/123

[Monitor CI/CD checks...]
Waiting for checks to complete...
All checks passed: ✓ lint, ✓ test, ✓ build

Ask: "Ready to merge this PR into main?"
User answers: "Yes, merge now"

[Execute: gh pr merge --merge]
PR #123 merged successfully into main

[Stop Docker environment...]
[Verify current location]
[Get project name from directory]
[Execute: docker compose -p "feature-add-user-authentication" down]
Docker containers for this branch stopped successfully
[Execute: docker compose -p "feature-add-user-authentication" ps]
No containers running for this project

[Switch to main...]
[Execute: git fetch origin]
[Execute: git checkout main]
[Execute: git pull origin main]
Now on branch: main

[Clean up...]
[Execute: git branch -d feature/add-user-authentication]
Local branch deleted
[Check remote: git ls-remote --heads origin feature/add-user-authentication]
Remote branch already deleted by GitHub
[Execute: git fetch --prune]

Done!
- PR #123 merged into main
- Docker containers stopped
- Branches cleaned up
- Current branch: main
- Ready for next task
```
