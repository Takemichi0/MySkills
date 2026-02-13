---
name: start-worktree
description: Create a new git worktree for starting work in a new terminal without affecting existing worktrees
user-invocable: true
model: haiku
allowed-tools: Bash(git *), Bash(cd *), Bash(pnpm *), Bash(docker *)
---

# Git Worktree Creation for New Terminal Session

Create a new git worktree in a separate directory when starting work in a new terminal. This allows you to work on a new branch without affecting any existing worktrees or branches.

## Workflow

### 1. Confirm Base Branch
- Get the current branch name: `git branch --show-current`
- Use AskUserQuestion: "You are currently on branch '<current-branch>'. Which branch should be the base for the new worktree?"
  - Options:
    - "This branch (<current-branch>)"
    - "main"
    - "Cancel"
- If user chooses "Cancel": STOP execution
- If user chooses "This branch": use current branch as base
- If user chooses "main": use `origin/main` as base (no checkout needed)

### 2. Get Current Directory Name and Generate New Directory Name
- Get the current directory name (e.g., "my-project")
- Generate a new directory name with sequential numbering (e.g., "my-project2", "my-project3")
- Check parent directory for existing numbered directories and find the next available number
- The new directory will be created at `../<new-directory-name>`

### 3. Ask User About New Work
- Use AskUserQuestion tool: "What kind of work do you want to do in the new branch?"
- Generate an appropriate branch name based on the answer:
  - For features: `feature/<descriptive-name>`
  - For bug fixes: `fix/<descriptive-name>`
  - For experiments: `experiment/<descriptive-name>`
  - Use kebab-case for the descriptive part

### 4. Determine Base Branch
- If user chose "main" in step 1: use `origin/main` as the base branch
- If user chose "This branch" in step 1:
  - Check if current branch has a remote tracking branch: `git rev-parse --abbrev-ref @{u}`
  - Use the remote tracking branch as the base (e.g., `origin/feature/user-profile`)
  - If no remote tracking branch exists, use `origin/main` as fallback

### 5. Create New Branch (Without Checking Out)
- Create the new branch based on the remote tracking branch: `git branch <new-branch-name> <remote-tracking-branch>`
- This creates the branch but does NOT checkout to it
- The current working directory remains on its original branch with all changes intact

### 6. Create New Worktree
- Execute: `git worktree add ../<new-directory-name> <new-branch-name>`
- This creates a new worktree in the new directory, checked out to the new branch
- The new worktree starts with a clean state based on the remote branch

### 7. Move to New Directory
- Change directory to the new worktree: `cd ../<new-directory-name>`
- Confirm the move was successful

### 8. Verify Setup
- Run `git branch --show-current` to confirm we're on the new branch
- Run `git status` to confirm clean state

### 9. Install Dependencies
- Execute: `pnpm install`
- This installs node_modules which are not included in the worktree (gitignored)

### 10. Start Docker Environment
- Calculate port numbers based on directory suffix:
  - Extract number from directory name (e.g., "my-app2" -> 2, "my-app3" -> 3)
  - Calculate app port: 3000 + number (e.g., my-app2 -> 3001, my-app3 -> 3002)
  - Calculate postgres port: 5432 + number (e.g., my-app2 -> 5433, my-app3 -> 5434)
- Generate project name from directory name:
  - Remove hyphens from directory name (e.g., "my-app2" -> "myapp2")
  - This ensures unique container names per worktree
- Inform user: "Starting Docker environment with COMPOSE_PROJECT_NAME=<project-name> PORT=<app-port> POSTGRES_PORT=<postgres-port>..."
- Execute: `COMPOSE_PROJECT_NAME=<project-name> PORT=<app-port> POSTGRES_PORT=<postgres-port> docker compose up -d --build`
- Inform user of the successful setup with summary:
  - New worktree location
  - Branch name
  - Docker URL with port number

## Important Notes

- The new worktree starts with a clean state based on the remote branch
- Each worktree is independent - you can have multiple terminals working on different branches simultaneously
- Files/folders in `.gitignore` (node_modules, .env, .claude/settings.local.json, build artifacts, etc.) are NOT included in the new worktree.
- Docker environments run independently in each worktree:
  - Different app port numbers (PORT) to avoid port conflicts
  - Different postgres port numbers (POSTGRES_PORT) to avoid database port conflicts
  - Different project names (COMPOSE_PROJECT_NAME) to avoid container name conflicts

## Example Flow

```
Original directory: ~/projects/my-app on branch feature/user-profile
Has uncommitted changes: src/profile.tsx (modified)
Has unpushed commits: 2 commits ahead of origin/feature/user-profile

User runs: /start-worktree

[Check current branch: feature/user-profile]
Ask: "You are currently on branch 'feature/user-profile'. Which branch should be the base for the new worktree?"
User answers: "This branch (feature/user-profile)"

[Check current directory: ~/projects/my-app]
[Generate new directory: ~/projects/my-app2]

Ask: "What kind of work do you want to do in the new branch?"
User answers: "Add settings page"
Generated branch name: feature/add-settings

[Get remote tracking branch: origin/feature/user-profile]
[Execute: git branch feature/add-settings origin/feature/user-profile]
[Execute: git worktree add ../my-app2 feature/add-settings]
[Change directory to ../my-app2]
[Verify setup]

[Execute: pnpm install]

[Calculate ports: my-app2 -> 2 -> PORT=3001, POSTGRES_PORT=5433]
[Generate project name: my-app2 -> myapp2]
Starting Docker environment with COMPOSE_PROJECT_NAME=myapp2 PORT=3001 POSTGRES_PORT=5433...
[Execute: COMPOSE_PROJECT_NAME=myapp2 PORT=3001 POSTGRES_PORT=5433 docker compose up -d --build]

Done! Now working in ~/projects/my-app2 on branch feature/add-settings
Docker is now running on http://localhost:3001

Result:
- ~/projects/my-app: still on branch feature/user-profile
  - src/profile.tsx still has uncommitted changes
  - 2 unpushed commits still present
  - Docker running on http://localhost:3000 (COMPOSE_PROJECT_NAME=myapp, POSTGRES_PORT=5432)
- ~/projects/my-app2: on branch feature/add-settings
  - Clean state from origin/feature/user-profile (before the unpushed commits)
  - Docker running on http://localhost:3001 (COMPOSE_PROJECT_NAME=myapp2, POSTGRES_PORT=5433)
  - Can work independently without affecting my-app (separate containers, separate databases)
```
