---
name: end-session
description: Complete current task, check TODO.md, commit, push, and optionally start new session
user-invocable: true
model: sonnet
allowed-tools: Bash(git *)
---

# Session Completion Workflow

Complete your current task by checking and updating TODO.md, committing changes, pushing to remote, and closing the session.

## Task Management Philosophy

This workflow follows a 1-task-1-session management approach:
- Each Claude Code session is dedicated to completing a single task
- When a task is complete, this skill helps finalize the work
- After committing and pushing, start a new session for the next task
- This approach keeps sessions focused and organized

## Workflow

### 1. Verify Current Branch
1. **Check current branch**:
   - Execute: `git branch --show-current`
   - If on `main` or `master`:
     - Use AskUserQuestion: "You are currently on the main/master branch. This is unusual for task work. Is this correct?"
       - Options: "Yes, continue" or "No, cancel"
     - If user says no: STOP execution and suggest they switch to a feature branch

### 2. Gather Context
1. **Gather context** (run in parallel):
   - `git status` to see all changes (never use -uall flag)
   - `git diff` to see the actual changes
   - `git log --oneline -10` to see recent commit message style

### 3. Check and Update TODO.md
1. **Check and update TODO.md**:
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

### 4. Check and Update CLAUDE.md and Other Documentation
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

### 5. Analyze Changes and Create Commit
Follow the CLAUDE.md git commit protocol:

1. **Analyze and draft commit message**:
   - Summarize the nature of changes (new feature, enhancement, bug fix, refactoring, etc.)
   - Use appropriate prefix: "add" for new features, "update" for enhancements, "fix" for bug fixes
   - Keep it concise and in English (one-liner)
   - Focus on what changed, not on implementation details
   - Add co-authored-by line

2. **Commit changes**:
   - Commit per file as a general rule. Stage and commit each file individually to keep git history granular and easy to revert.
   - If multiple files are tightly coupled and form a single logical change, they can be committed together.
   - Stage changes: `git add <file>` (per file, not `git add .`)
   - Commit with message using HEREDOC:
     ```bash
     git commit -m "$(cat <<'EOF'
     <commit message>

     Co-Authored-By: Claude Sonnet 4.5 <noreply@anthropic.com>
     EOF
     )"
     ```
   - Verify: `git status`

### 6. Push to Remote
- Get current branch: `git branch --show-current`
- Push with upstream tracking: `git push -u origin <current-branch>`
- Verify push was successful

### 7. Session Management
- Display summary:
  - Changes committed and pushed to origin/<branch-name>
  - TODO.md updated (if applicable)
  - Ready for next task

### 8. Exit Session
- Inform the user: "Session work is complete. Please run `/exit` or `/clear` to end the session."
- Do NOT attempt to execute `/exit` internally (it cannot be run from within a skill)

## Important Notes

- This command is for completing individual tasks in a 1-task-1-session workflow
- Commits are always made to the current branch, then pushed to origin
- If on main/master branch, user confirmation is required before proceeding
- TODO.md is automatically updated to mark completed tasks
- All commit messages are concise one-liners in English
- At the end of the workflow, the user is prompted to manually run `/exit` or `/clear` to close the session

## Example Flow

```
User runs: /end-session

[Check current branch: feature/add-login]
Branch is not main/master, proceeding...

[Gather context...]
[Execute: git status]
[Execute: git diff]
[Execute: git log --oneline -10]

[Analyze changes and draft commit message...]
Commit message: "add login form with validation"

[Check TODO.md...]
TODO.md found at project root
Found matching task: "- [ ] Implement login form"
Marking as completed: "- [x] Implement login form"

[Check documentation updates...]
Analyzing changes: New login form feature added
Ask: "The work included significant changes. Should I update CLAUDE.md and other documentation files to reflect these changes?"
User answers: "Yes, update documentation"

[Update CLAUDE.md...]
Found CLAUDE.md at project root
Updating to reflect new login form feature
CLAUDE.md updated successfully

[Commit changes...]
[Execute: git add .]
[Execute: git commit -m "add login form with validation"]
Commit created successfully

[Push to remote...]
[Execute: git push -u origin feature/add-login]
Pushed successfully to origin/feature/add-login

[Display summary...]
- Committed: "add login form with validation"
- Pushed to: origin/feature/add-login
- TODO.md updated
- Session work is complete. Please run `/exit` or `/clear` to end the session.
```
