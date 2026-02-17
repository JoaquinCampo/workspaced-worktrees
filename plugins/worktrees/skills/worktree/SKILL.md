---
name: worktree
description: Create an isolated worktree workspace for parallel branch development. Reads config from .claude/worktree.json.
argument-hint: feature/my-feature-name
---

# Create Worktree Workspace

Create an isolated workspace with git worktrees for each repo, so a Claude Code instance can work on a feature branch in parallel without affecting the main checkout.

## Input

Branch name: `$ARGUMENTS`

If no argument provided, ask the user for the branch name.

## Step 1: Read config

Read `.claude/worktree.json` from the project root.

If the file does **not** exist:

1. Scan the current directory for subdirectories that are git repos (`[ -d "$dir/.git" ]`)
2. Create a template `.claude/worktree.json` populated with the discovered repos:

```json
{
  "repos": [
    {
      "name": "<detected-dir>",
      "port": 8000,
      "install": "<FILL IN: e.g. npm install, uv sync, pnpm install>",
      "start": "<FILL IN: e.g. npm run dev --port {port}>",
      "env_files": [".env"],
      "env_patches": {}
    }
  ]
}
```

3. Print:
```
Created .claude/worktree.json with detected repos.
Please review and fill in the install/start commands, then run /worktree again.
```

4. **Stop here.** Do not create any worktrees.

---

If the file **does** exist, proceed with the steps below.

## Step 2: Derive the slug

```bash
SLUG=$(echo "$ARGUMENTS" | tr '/' '-')
WORKSPACE="../$(basename $(pwd))-wt-${SLUG}"
```

## Step 3: Create the workspace root

```bash
mkdir -p "$WORKSPACE"
```

## Step 4: Assign unique ports

For each repo in the config, calculate its port:

```bash
OFFSET=$(cd <repo.name> && git worktree list | wc -l | tr -d ' ')
```

The port for each repo is `repo.port + OFFSET`.

If any assigned port is busy (`lsof -i :<port>`), increment all ports by 1 and check again.

Keep a map of `repo.name → assigned_port` for use in env patching.

## Step 5: Create worktrees

For each repo in the config:

```bash
cd <repo.name>
git worktree add "${WORKSPACE}/<repo.name>" -b "$ARGUMENTS" 2>/dev/null \
  || git worktree add "${WORKSPACE}/<repo.name>" "$ARGUMENTS"
```

If creation fails because the branch exists remotely but not locally:
```bash
git fetch origin "$ARGUMENTS"
```
Then retry.

## Step 6: Copy root config

Copy the root `CLAUDE.md` and `.claude/` directory so the workspace inherits shared commands, skills, and rules:

```bash
cp CLAUDE.md "$WORKSPACE/" 2>/dev/null || true
cp -r .claude "$WORKSPACE/" 2>/dev/null || true
```

## Step 7: Install dependencies

For each repo in the config:

```bash
cd "${WORKSPACE}/<repo.name>" && <repo.install>
```

## Step 8: Copy and patch environment files

For each repo in the config:

1. Copy each file listed in `repo.env_files` from the original repo into the worktree:
   ```bash
   cp <repo.name>/<env_file> "${WORKSPACE}/<repo.name>/<env_file>" 2>/dev/null || true
   ```

2. Apply `repo.env_patches`: for each key-value pair, find and replace that key's value in the worktree's env file. Values can contain `{<repo_name>.port}` placeholders — replace them with the actual assigned port for that repo.

## Step 9: Report

Print a summary:

```
Workspace ready for branch: $ARGUMENTS
  Root: <WORKSPACE>/
```

For each repo:
```
  <repo.name>:
    Path: <WORKSPACE>/<repo.name>
    Port: <assigned_port>
    Start: <repo.start with {port} replaced>
```

Then print:
```
Launch agent:
  cd <WORKSPACE> && claude --dangerously-skip-permissions
```

## Cleanup

Print cleanup instructions:

For each repo:
```bash
cd <repo.name> && git worktree remove "<WORKSPACE>/<repo.name>"
```

Then:
```bash
rm -rf "<WORKSPACE>"
```
