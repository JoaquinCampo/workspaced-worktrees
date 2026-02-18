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
      "env_patches": {},
      "cache_dirs": []
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

## Step 4: Detect portless and assign service endpoints

Check if [portless](https://github.com/vercel-labs/portless) is installed:

```bash
command -v portless &>/dev/null
```

### If portless is installed → Portless mode

For each repo, derive a **portless service name**:

```
SERVICE_NAME="<repo.name>-${SLUG}"
```

Example: repo `backend-parsanai` on branch `feature/auth` → `backend-parsanai-feature-auth`

Build a URL map for env patching:
```
repo.name → http://<SERVICE_NAME>.localhost:1355
```

No port assignment, offset counting, or conflict resolution needed. Skip the fallback section below.

### If portless is NOT installed → Fallback mode

For each repo in the config, calculate its port:

```bash
OFFSET=$(cd <repo.name> && git worktree list | wc -l | tr -d ' ')
```

The port for each repo is `repo.port + OFFSET`.

If any assigned port is busy (`lsof -i :<port>`), increment all ports by 1 and check again.

Build a URL map for env patching:
```
repo.name → http://localhost:<assigned_port>
```

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

## Step 7: Generate worktree rules

Create `${WORKSPACE}/.claude/rules/worktree.md` so the agent automatically knows its environment. This file is auto-loaded into context.

Make sure the `.claude/rules/` directory exists:
```bash
mkdir -p "${WORKSPACE}/.claude/rules"
```

Generate the file with the following content (replace all placeholders with actual values):

### Portless mode

```markdown
# Worktree Workspace

You are working in an isolated worktree workspace on branch `<branch_name>`.

## Servers

Start all servers:
\`\`\`bash
./start.sh
\`\`\`

Stop all servers:
\`\`\`bash
./stop.sh
\`\`\`

<for each repo>
### <repo.name>
- Path: <repo.name>/
- URL: http://<service_name>.localhost:1355
- Start individually: cd <repo.name> && portless <service_name> <repo.start with {port} replaced by \$PORT>
<end for>

## Rules

- **No migrations.** Never run migration tools (alembic, prisma migrate, knex migrate, etc.) or any command that auto-runs migrations on startup. The database is shared across all worktrees.
- **Schema changes = blocker.** If your feature requires database schema changes, STOP and report back to the user.
- **Don't push** to remote unless explicitly asked.
- **Commit freely** to your feature branch — commits stay local until pushed.

## Cleanup

When done, the user can run `./cleanup.sh` from the workspace root to stop servers, remove worktrees, and delete this workspace.
```

### Fallback mode

```markdown
# Worktree Workspace

You are working in an isolated worktree workspace on branch `<branch_name>`.

## Servers

Start all servers:
\`\`\`bash
./start.sh
\`\`\`

Stop all servers:
\`\`\`bash
./stop.sh
\`\`\`

<for each repo>
### <repo.name>
- Path: <repo.name>/
- Port: <assigned_port>
- Start individually: cd <repo.name> && <repo.start with {port} replaced>
<end for>

## Rules

- **No migrations.** Never run migration tools (alembic, prisma migrate, knex migrate, etc.) or any command that auto-runs migrations on startup. The database is shared across all worktrees.
- **Schema changes = blocker.** If your feature requires database schema changes, STOP and report back to the user.
- **Don't push** to remote unless explicitly asked.
- **Commit freely** to your feature branch — commits stay local until pushed.

## Cleanup

When done, the user can run `./cleanup.sh` from the workspace root to stop servers, remove worktrees, and delete this workspace.
```

## Step 8: Copy cached dependency directories

For each repo in the config, if `repo.cache_dirs` is defined and non-empty, clone each directory from the original repo into the worktree **before** installing. This avoids downloading everything from scratch.

On macOS (APFS), use `cp -Rc` for instant copy-on-write clones:
```bash
cp -Rc <repo.name>/<cache_dir> "${WORKSPACE}/<repo.name>/<cache_dir>"
```

On Linux, use hardlinks for speed:
```bash
cp -al <repo.name>/<cache_dir> "${WORKSPACE}/<repo.name>/<cache_dir>"
```

Skip any that don't exist in the original.

**Hint for users configuring `cache_dirs`:** APFS clones and hardlinks are fast for directories with few large files (e.g., `.venv`, `vendor/`), but slow for directories with many small files (e.g., `node_modules`). Package managers with global caches (pnpm, yarn berry, uv) often make a fresh install fast enough that cloning is unnecessary. When in doubt, leave `cache_dirs` empty and let the package manager handle it.

## Step 9: Install dependencies

For each repo in the config:

```bash
cd "${WORKSPACE}/<repo.name>" && <repo.install>
```

This should be fast since cached directories were already copied.

## Step 10: Copy and patch environment files

For each repo in the config:

1. Copy each file listed in `repo.env_files` from the original repo into the worktree:
   ```bash
   cp <repo.name>/<env_file> "${WORKSPACE}/<repo.name>/<env_file>" 2>/dev/null || true
   ```

2. Apply `repo.env_patches`: for each key-value pair, find and replace that key's value in the worktree's env file. Values support two kinds of placeholders:

   - **`{<repo_name>.url}`** — resolves to the full URL from the URL map built in Step 4. Works in both modes. **Recommended.**
   - **`{<repo_name>.port}`** — resolves to the assigned port number in fallback mode. In portless mode, if the value matches the pattern `http://localhost:{<repo_name>.port}`, replace the **entire pattern** with the portless URL from the URL map. This ensures backward compatibility with existing configs.

## Step 11: Generate workspace scripts

Create executable scripts in the workspace root so agents (or users) can start/stop servers without knowing the details.

### Portless mode

#### `start.sh`

```bash
#!/usr/bin/env bash
set -e
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

<for each repo>
echo "Starting <repo.name> → http://<service_name>.localhost:1355"
(cd "$SCRIPT_DIR/<repo.name>" && portless <service_name> <repo.start with {port} replaced by \$PORT>) &
<end for>

echo ""
echo "All servers started."
echo "Run ./stop.sh to shut them down."
wait
```

#### `stop.sh`

```bash
#!/usr/bin/env bash

<for each repo>
echo "Stopping <service_name>..."
pkill -f "portless <service_name>" 2>/dev/null || true
<end for>

echo "All servers stopped."
```

### Fallback mode

#### `start.sh`

```bash
#!/usr/bin/env bash
set -e
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

# Start each repo's server in the background
<for each repo>
echo "Starting <repo.name> on port <assigned_port>..."
(cd "$SCRIPT_DIR/<repo.name>" && <repo.start with {port} replaced>) &
<end for>

echo ""
echo "All servers started. PIDs logged above."
echo "Run ./stop.sh to shut them down."
wait
```

#### `stop.sh`

```bash
#!/usr/bin/env bash

<for each repo>
echo "Stopping port <assigned_port>..."
lsof -ti :<assigned_port> | xargs kill 2>/dev/null || true
<end for>

echo "All servers stopped."
```

### Both modes: `cleanup.sh`

```bash
#!/usr/bin/env bash
set -e
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
PARENT_DIR="$(dirname "$SCRIPT_DIR")"
WORKSPACE_NAME="$(basename "$SCRIPT_DIR")"

# Stop any running servers
"$SCRIPT_DIR/stop.sh" 2>/dev/null || true

# Remove worktrees
<for each repo>
echo "Removing worktree <repo.name>..."
git -C "$PARENT_DIR/<original_project>/<repo.name>" worktree remove "$SCRIPT_DIR/<repo.name>" 2>/dev/null || true
<end for>

# Remove workspace directory
echo "Removing workspace..."
rm -rf "$SCRIPT_DIR"
echo "Cleanup complete."
```

Make all scripts executable:
```bash
chmod +x "${WORKSPACE}/start.sh" "${WORKSPACE}/stop.sh" "${WORKSPACE}/cleanup.sh"
```

## Step 12: Report

### Portless mode

```
Workspace ready for branch: $ARGUMENTS
  Root: <WORKSPACE>/

<for each repo>
  <repo.name>:
    Path: <WORKSPACE>/<repo.name>
    URL:  http://<service_name>.localhost:1355
<end for>

Scripts:
  ./start.sh    — start all servers
  ./stop.sh     — stop all servers
  ./cleanup.sh  — stop servers, remove worktrees, delete workspace

Launch agent:
  cd <WORKSPACE> && claude --dangerously-skip-permissions
```

### Fallback mode

```
Workspace ready for branch: $ARGUMENTS
  Root: <WORKSPACE>/

<for each repo>
  <repo.name>:
    Path: <WORKSPACE>/<repo.name>
    Port: <assigned_port>
<end for>

Scripts:
  ./start.sh    — start all servers
  ./stop.sh     — stop all servers
  ./cleanup.sh  — stop servers, remove worktrees, delete workspace

Tip: Install portless for stable named URLs → npm i -g portless
     https://github.com/vercel-labs/portless

Launch agent:
  cd <WORKSPACE> && claude --dangerously-skip-permissions
```
