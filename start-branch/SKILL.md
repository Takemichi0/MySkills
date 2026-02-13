---
name: start-branch
description: Create a new branch from origin/main for starting a new task
user-invocable: true
model: haiku
allowed-tools: Bash(git *), Bash(docker *), Bash(lsof *), Bash(grep *), Bash(netstat *), Bash(echo *)
---

# Create New Branch from Origin/Main

Create a new branch from origin/main for starting a new task. This is intended for use when starting a larger task that requires a dedicated branch.

## Workflow

### 1. Verify Current State
- Check if you are in a git repository: `git rev-parse --is-inside-work-tree`
- If not in a git repository, inform the user and STOP execution

### 2. Fetch Latest Changes
- Fetch the latest changes from remote: `git fetch origin`
- This ensures we have the latest information about origin/main

### 3. Verify Local and Remote Main are in Sync
- Compare local main and origin/main:
  - Get local main commit: `git rev-parse main`
  - Get origin/main commit: `git rev-parse origin/main`
- If they are the same: Proceed to next step
- If they differ: Inform user that local main is NOT in sync with origin/main and STOP execution
  - The assumption is that local main and origin/main should be the same
  - User should resolve this manually (e.g., `git pull origin main`) before running this skill

### 4. Ask User About New Work
- Use AskUserQuestion tool: "What kind of work do you want to do in the new branch?"
- Generate an appropriate branch name based on the answer:
  - For features: `feature/<descriptive-name>`
  - For bug fixes: `fix/<descriptive-name>`
  - For experiments: `experiment/<descriptive-name>`
  - Use kebab-case for the descriptive part
- Then ask user to confirm the generated branch name with option to modify it

### 5. Create New Branch
- Execute: `git checkout -b <new-branch-name> origin/main`
- This creates the new branch from origin/main and switches to it
- Works regardless of which branch you are currently on

### 6. Verify Setup
- Run `git branch --show-current` to confirm we're on the new branch
- Run `git status` to confirm clean state

### 7. Find Available Port
- Check for ports in use starting from 3000: `lsof -iTCP -sTCP:LISTEN -n -P | grep :3000`
- If port 3000 is in use, check 3001, then 3002, etc. until an available port is found
- Use the first available port starting from 3000

### 8. Start Docker Environment
- Inform user: "Starting Docker environment on PORT=<available-port>..."
- Execute: `PORT=<available-port> docker compose up -d --build`
- Inform user of the successful setup with summary:
  - Branch name
  - Docker URL (http://localhost:<available-port>)

## Important Notes

- This is typically used when starting work in a new terminal
- The new branch will be created directly from origin/main
- Local main must be in sync with origin/main before proceeding
- Works regardless of which branch you are currently on
- Any uncommitted changes in your current branch are not affected
- Docker environment is automatically started after branch creation
- Port is automatically selected starting from 3000, avoiding conflicts with already running containers

## Example Flow

```
User runs: /start-branch

[Check git repository: OK]
[Execute: git fetch origin]
[Compare: git rev-parse main vs origin/main]
  main: abc123
  origin/main: abc123
  Result: In sync

Ask: "What kind of work do you want to do in the new branch?"
User answers: "Add user authentication feature"
Generated branch name: feature/add-user-authentication

Ask: "Create branch 'feature/add-user-authentication'?"
  Options:
  - "Yes, create this branch"
  - "No, let me specify a different name"
User answers: "Yes, create this branch"

[Execute: git checkout -b feature/add-user-authentication origin/main]
[Verify: git branch --show-current]
  Current branch: feature/add-user-authentication
[Execute: git status]
  On branch feature/add-user-authentication
  nothing to commit, working tree clean

[Check port availability]
[Execute: lsof -iTCP -sTCP:LISTEN -n -P | grep :3000]
  Result: Port 3000 is in use
[Execute: lsof -iTCP -sTCP:LISTEN -n -P | grep :3001]
  Result: Port 3001 is available

Starting Docker environment on PORT=3001...
[Execute: PORT=3001 docker compose up -d --build]

Done! Now working on branch feature/add-user-authentication
Docker is now running on http://localhost:3001
```
