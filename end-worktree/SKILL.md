---
name: end-worktree
description: Complete work in current worktree by updating TODO.md, creating PR, merging after checks pass, and cleaning up
user-invocable: true
model: sonnet
allowed-tools: Bash(git *), Bash(gh pr create *), Bash(gh pr checks *), Bash(gh pr view *), Bash(gh api *), Bash(cd *), Bash(pwd *), Bash(docker *)
---

# Git Worktree Completion and Cleanup

Complete your work in the current worktree by checking and updating TODO.md, committing changes, creating a pull request to merge back into the original branch (where you ran `/start-worktree` from), waiting for CI/CD checks, merging, returning to the original directory, and cleaning up the worktree and branches.

## Workflow

### 1. Verify Current State
- Confirm we're in a worktree (not the main working directory)
- Get current branch name: `git branch --show-current`
- Get current worktree path: `git worktree list` and `pwd`
- Verify there are changes to commit: `git status --porcelain`

### 2. Analyze Changes and Create Commit
Follow the CLAUDE.md git commit protocol:

1. **Gather context** (run in parallel):
   - `git status` to see all changes (never use -uall flag)
   - `git diff` to see the actual changes
   - `git log --oneline` to see recent commit message style

2. **Check and update TODO.md** (before staging):
   - Check if TODO.md exists in the project root
   - If it exists:
     - Read TODO.md content
     - Analyze current changes (from git diff and git status) to understand what work was done
     - Search TODO.md for items that match the completed work
     - For matching items:
       - Mark them as completed (add `[x]` or strikethrough, depending on the format used)
       - Update TODO.md with the changes
     - If no matching items found, skip this step
   - If TODO.md doesn't exist, skip this step

3. **Check and Update CLAUDE.md and Other Documentation**:
   1. **Analyze if documentation updates are needed**:
      - Review the completed work and changes made
      - Determine if there were:
        - Major requirement changes
        - New feature implementations
        - Architectural changes
        - Other significant modifications that should be documented

   2. **If updates are needed**:
      - Use AskUserQuestion: "The work included significant changes. Should I update CLAUDE.md and other documentation files to reflect these changes?"
        - Options: "Yes, update documentation" or "No, skip documentation updates"
      - If user says no: Skip to next step

   3. **If user approves documentation updates**:
      - Find all CLAUDE.md files in the project hierarchy
      - Start with the most specific (deepest) CLAUDE.md file
      - Update it to reflect the changes made
      - If the changes are significant, propagate updates to parent CLAUDE.md files up to the project root
      - Check and update other relevant .md files as needed
      - Ensure consistency between CLAUDE.md and TODO.md

4. **Analyze and draft commit message**:
   - Summarize the nature of changes (new feature, enhancement, bug fix, refactoring, etc.)
   - Use appropriate prefix: "add" for new features, "update" for enhancements, "fix" for bug fixes
   - Keep it concise and in English (one-liner)
   - Focus on what changed, not on implementation details
   - Add co-authored-by line

5. **Commit changes**:
   - Stage changes: `git add .` (this includes TODO.md and CLAUDE.md if they were updated)
   - Commit with message using HEREDOC:
     ```bash
     git commit -m "$(cat <<'EOF'
     <commit message>

     Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
     EOF
     )"
     ```
   - Verify: `git status`

### 3. Push to Remote
- Push with upstream tracking: `git push -u origin <current-branch>`
- Verify push was successful

### 4. Create Pull Request
1. **Determine base branch (where the branch was forked from)**:
   - Find the closest parent branch by comparing commit counts with all remote branches
   - Algorithm:
     1. Get current branch name: `git branch --show-current`
     2. Get all remote branches: `git branch -r`
     3. For each remote branch (excluding current branch and HEAD):
        - Count commits ahead: `git log origin/<branch>..HEAD --oneline | wc -l`
     4. The branch with the smallest non-zero commit count is the parent
        - This works because the parent branch shares the most history with the current branch
        - Example: if `origin/main..HEAD` has 26 commits but `origin/feature/parent..HEAD` has 12 commits, the parent is `feature/parent`
   - If all branches have 0 commits ahead, fall back to main or master
   - The PR will merge this worktree's branch back into the original branch (NOT always main)

2. **Analyze commit history for PR**:
   - `git log origin/<base-branch>...HEAD` to see all commits in this PR
   - `git diff origin/<base-branch>...HEAD` to see full changes

3. **Generate PR title and body**:
   - Title: Short summary (under 70 characters) based on commit messages
   - Body: Include summary of changes and test plan
   - Format:
     ```markdown
     ## Summary
     <bullet points of changes>

     ## Test plan
     <checklist of testing steps>

     Generated with Claude Code
     ```

4. **Create PR**:
   ```bash
   gh pr create --base <base-branch> --title "<title>" --body "$(cat <<'EOF'
   <body content>
   EOF
   )"
   ```
   - The `--base` flag specifies which branch this PR will merge into
   - Capture PR URL from output

### 5. Monitor CI/CD Checks
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

4. **Check results**:
   - If all checks pass: proceed to merge
   - If any checks fail:
     - Display failed check names and details
     - Show PR URL for investigation
     - Use AskUserQuestion: "CI/CD checks failed. Do you want to ignore the failures and proceed with merge anyway?"
       - Options: "Yes, ignore and merge" or "No, stop and fix issues"
     - If user says "Yes, ignore and merge": proceed to merge confirmation step
     - If user says "No, stop and fix issues":
       - STOP execution
       - Inform user they can fix issues and run `/end-worktree` again

### 6. Confirm Merge with User
- Display PR summary:
  - PR title and URL
  - Number of commits
  - Files changed
  - All checks passed
- Use AskUserQuestion: "All checks have passed. Ready to merge this PR?"
  - Options: "Yes, merge now" or "No, cancel"
- If user says no: STOP and inform them they can merge manually later

### 7. Merge Pull Request
- Execute: `gh pr merge --merge` (create merge commit)
- Verify merge was successful
- GitHub typically auto-deletes the remote branch after merge (if configured)

### 8. Stop Docker Environment
1. **Verify current location**:
   - Confirm we're still in the worktree directory
   - `pwd` to check current path

2. **Stop Docker containers for this worktree only**:
   - Get directory name: `DIR_NAME=$(basename "$(pwd)")`
   - Generate project name by removing hyphens (same as start-worktree): `PROJECT_NAME=$(echo "$DIR_NAME" | tr -d '-')`
   - `docker compose -p "$PROJECT_NAME" down`
   - This stops and removes only containers associated with this worktree
   - Other worktrees running their own containers are not affected

3. **Verify cleanup**:
   - `docker compose -p "$PROJECT_NAME" ps` to confirm no containers are running for this worktree

### 9. Clean Up Remote Resources
These operations can be executed from any directory, so they are safe to run while still in the worktree.

1. **Verify remote branch deletion**:
   - `git ls-remote --heads origin <branch-name>` to confirm remote branch was deleted
   - GitHub automatically deletes the remote branch after PR merge
   - If by any chance it still exists: `git push origin --delete <branch-name>` to delete it

2. **Clean up remote tracking references**:
   - `git fetch --prune` to remove stale remote-tracking branches

### 10. Prepare Cleanup Information
Gather the necessary information for the user to complete cleanup manually.

1. **Get original directory path**:
   - `git worktree list | head -n1 | awk '{print $1}'`
   - This is the main worktree where the user should return

2. **Get current worktree path**:
   - `pwd` to get the full path of the current worktree

3. **Get current branch name**:
   - `git branch --show-current`

### 11. Final Summary and User Instructions
Display a summary of completed actions and provide clear instructions for the remaining cleanup steps.

**Summary of completed actions**:
- PR merged successfully into <base-branch>
- TODO.md and CLAUDE.md updated (if applicable)
- Docker containers stopped
- Remote branch deleted (if it existed)
- Remote tracking references cleaned up

**User instructions for final cleanup**:
```
This skill has completed. To finish cleanup, please run the following commands:

cd <original-directory-path>
git worktree remove <current-worktree-path>
git pull
git branch -d <branch-name>
```

**Important notes**:
- The `cd` command is necessary because Claude Code cannot change the system's working directory
- `git worktree remove` will delete the worktree directory, so it must be run from outside that directory
- After running these commands, you will be back in <original-directory-path> on branch <base-branch>

## Important Notes

- This command should only be run from within a worktree (not the main directory)
- **The PR merges into the original branch** (where you ran `/start-worktree` from), which could be:
  - `main` if you started from main
  - `feature/user-profile` if you started from a feature branch still in development
  - This is NOT always main - it depends on where the worktree was created from
- All changes must be committed before the command completes
- CI/CD checks must pass before merging (or user can choose to ignore failures)
- User confirmation is required before merge
- Docker containers are stopped before cleanup (assumes Docker environment was started with `/start-worktree`)
- Remote branch is deleted automatically by GitHub after merge (or manually if needed)
- **Worktree and local branch cleanup requires manual user action** after the skill completes
- The skill will provide clear instructions with the exact commands to run
- The command will stop at any failure point and inform the user

## Why Merge Commit Instead of Rebase?

This skill uses `--merge` (create merge commit) instead of `--rebase` for the following reasons:

- **Reflects actual development history**: Merge commits preserve the true timeline of how the work was done, showing when the feature branch diverged and when it was integrated back
- **Same conflict resolution**: Both merge and rebase can encounter conflicts, so there's no advantage in conflict handling
- **Preserves context**: Merge commits make it easy to see all changes that were part of a feature in one place
- **Rebase is rarely necessary**: Unless there's a serious mistake in commit history that needs to be hidden, rebase doesn't provide meaningful benefits
- **Simpler and safer**: Merge commits don't rewrite history, making them safer and easier to understand for team collaboration

Use rebase only when you have a compelling reason, such as needing to remove sensitive data or fix a critical mistake in commit history. For normal development workflow, merge commits are preferred.

## Example Flow

```
User runs: /end-worktree

[Current directory: ~/projects/my-app3]
[Current branch: feature/add-settings]
[Worktree path: ~/projects/my-app3]
[Original directory: ~/projects/my-app on branch feature/user-profile]

[Check git status...]
Found changes to commit: 2 files modified

[Check TODO.md...]
TODO.md found, marking completed items...

[Check documentation updates...]
Analyzing changes: New settings page feature added
Ask: "The work included significant changes. Should I update CLAUDE.md and other documentation files to reflect these changes?"
User answers: "Yes, update documentation"

[Update CLAUDE.md...]
Found CLAUDE.md at project root
Updating to reflect new settings page feature
CLAUDE.md updated successfully

[Analyze and draft commit message...]
Commit message: "add settings page with user preferences"

[Commit and push...]
[Execute: git add .]
[Execute: git commit -m "add settings page with user preferences"]
[Execute: git push -u origin feature/add-settings]

[Determine base branch...]
[Compare commit counts with remote branches:]
  - origin/main..HEAD: 26 commits
  - origin/feature/user-profile..HEAD: 5 commits  <- smallest non-zero
  - origin/feature/add-settings..HEAD: 0 commits (current branch, skip)
[Base branch: feature/user-profile (closest parent)]

[Analyze commits for PR...]
[Execute: git log origin/feature/user-profile...HEAD]
PR title: "Add settings page"

[Execute: gh pr create --base feature/user-profile --title "..." --body "..."]
PR created: https://github.com/user/my-app/pull/45

[Monitor CI/CD checks...]
All checks passed: ✓ lint, ✓ test, ✓ build

Ask: "All checks have passed. Ready to merge this PR?"
User answers: "Yes, merge now"

[Execute: gh pr merge --merge]
PR #45 merged successfully (merged into feature/user-profile)

[Stop Docker environment...]
[Verify current location: ~/projects/my-app3]
[Get directory name: my-app3]
[Generate project name (remove hyphens): myapp3]
[Execute: docker compose -p "myapp3" down]
Docker containers for this worktree stopped successfully
[Execute: docker compose -p "myapp3" ps]
No containers running for this worktree

[Clean up remote resources...]
[Check remote branch: git ls-remote --heads origin feature/add-settings]
Remote branch already deleted by GitHub
[Execute: git fetch --prune]
Remote tracking references cleaned up

[Prepare cleanup information...]
[Execute: git worktree list | head -n1 | awk '{print $1}']
Original directory: ~/projects/my-app
[Execute: pwd]
Current worktree: ~/projects/my-app3
[Execute: git branch --show-current]
Branch name: feature/add-settings

Done! Summary:
- PR #45 merged into feature/user-profile
- TODO.md and CLAUDE.md updated
- Docker containers stopped
- Remote branch deleted
- Remote tracking references cleaned up

To finish cleanup, please run the following commands:

cd ~/projects/my-app
git worktree remove ~/projects/my-app3
git pull
git branch -d feature/add-settings

After running these commands, you will be in ~/projects/my-app on branch feature/user-profile.
Note: feature/user-profile itself may still be in development, not yet merged to main.
```
