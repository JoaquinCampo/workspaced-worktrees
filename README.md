# Workspaced Worktrees

A Claude Code plugin for parallel branch development in monorepos. Create isolated workspaces with auto-wired environments and generated scripts — so multiple agents (or humans) can work on different features simultaneously without conflicts.

Supports [portless](https://github.com/vercel-labs/portless) for stable named `.localhost` URLs, with automatic fallback to port-based assignment.

## Install

```
/plugin marketplace add JoaquinCampo/workspaced-worktrees
/plugin install worktrees@workspaced-worktrees
```

When prompted, choose **user** scope to make it available across all projects.

## Setup (per project)

Run from your monorepo root:

```
/worktrees:worktree feature/test
```

First run auto-detects your git repos and creates a template config at `.claude/worktree.json`:

```json
{
  "repos": [
    {
      "name": "backend",
      "port": 8000,
      "install": "uv sync",
      "start": "uv run uvicorn app.main:app --port {port}",
      "env_files": [".env"],
      "env_patches": {},
      "cache_dirs": [".venv"]
    },
    {
      "name": "frontend",
      "port": 3000,
      "install": "pnpm install",
      "start": "pnpm dev --port {port}",
      "env_files": [".env"],
      "env_patches": {
        "NEXT_PUBLIC_API_URL": "http://localhost:{backend.port}"
      },
      "cache_dirs": []
    }
  ]
}
```

Fill in the `install` and `start` commands for your stack, then commit the file. Your whole team gets the config.

### Config fields

| Field | Description |
|---|---|
| `name` | Subdirectory name (must be a git repo) |
| `port` | Base port number. Auto-incremented to avoid collisions. Only used when portless is not installed |
| `install` | Command to install dependencies |
| `start` | Command to start the dev server. `{port}` is replaced with the assigned port |
| `env_files` | Environment files to copy into the worktree |
| `env_patches` | Key-value pairs to patch in env files. Values support `{<repo_name>.url}` (recommended) and `{<repo_name>.port}` placeholders |
| `cache_dirs` | Directories to clone before installing (e.g., `.venv`). Leave empty for package managers with global caches (pnpm, yarn berry) |

## Usage

```
/worktrees:worktree feature/my-feature
```

This creates:

```
../my-project-wt-feature-my-feature/
├── .claude/
│   └── rules/
│       └── worktree.md     # auto-loaded agent context
├── CLAUDE.md                # inherited from project root
├── start.sh                 # start all servers
├── stop.sh                  # stop all servers
├── cleanup.sh               # tear down everything
├── backend/                 # git worktree on feature/my-feature
└── frontend/                # git worktree on feature/my-feature
```

Then launch an agent:

```bash
cd ../my-project-wt-feature-my-feature && claude --dangerously-skip-permissions
```

The agent automatically knows its ports, how to start servers, and the workspace rules — all via the generated `.claude/rules/worktree.md`.

## Portless integration

If [portless](https://github.com/vercel-labs/portless) is installed (`npm i -g portless`), workspaces automatically get stable named URLs instead of port numbers:

```
backend-feature-auth.localhost:1355      # instead of localhost:8001
frontend-feature-auth.localhost:1355     # instead of localhost:3001
```

No port assignment, offset counting, or conflict resolution needed. Cross-service env references (`NEXT_PUBLIC_API_URL`) resolve to portless URLs automatically.

If portless is not installed, the plugin falls back to port-based assignment — no changes needed.

## Parallel work

Each workspace gets unique endpoints, so you can run multiple features simultaneously:

**With portless:**
```
backend-feature-auth.localhost:1355          # agent 1
frontend-feature-auth.localhost:1355
backend-feature-dashboard.localhost:1355     # agent 2
frontend-feature-dashboard.localhost:1355
```

**Without portless:**
```
your-project/                           # main checkout — :8000, :3000
../your-project-wt-feature-auth/        # agent 1 — :8001, :3001
../your-project-wt-feature-dashboard/   # agent 2 — :8002, :3002
../your-project-wt-fix-upload-bug/      # agent 3 — :8003, :3003
```

## Generated scripts

| Script | Purpose |
|---|---|
| `./start.sh` | Start all servers with assigned ports |
| `./stop.sh` | Stop all servers |
| `./cleanup.sh` | Stop servers, remove worktrees, delete workspace |

## What agents know automatically

The generated `.claude/rules/worktree.md` tells agents:

- Which branch they're on
- Service URLs (portless) or port assignments and start commands for each repo
- To use `./start.sh` and `./stop.sh` for server management
- **No migrations** — database is shared across worktrees
- **Schema changes are a blocker** — report back to the user
- **Don't push** unless explicitly asked

## Cleanup

From the workspace:

```bash
./cleanup.sh
```

Or manually:

```bash
cd your-project/backend && git worktree remove ../your-project-wt-feature-auth/backend
cd your-project/frontend && git worktree remove ../your-project-wt-feature-auth/frontend
rm -rf ../your-project-wt-feature-auth
```

## Update

```
/plugin marketplace update workspaced-worktrees
```
