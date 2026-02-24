# Projects App


![projects screenshot](projects.png)
Read-only dashboard showing all git repositories under `~/git/`. Displays branch, last commit, dirty/clean status, ahead/behind counts, and remote URLs in a table.

## Ports

| Component | Port |
|-----------|------|
| Frontend  | 4007 |
| Backend   | 4008 |

## Backend

**File:** `apps/projects/backend/main.py`
**Stack:** FastAPI
**Dependencies:** `fastapi`, `uvicorn[standard]`

### Endpoints

#### `GET /api/projects`

Scans `~/git/` for non-hidden subdirectories. For each directory, runs git commands (with 5-second timeouts) to collect:

- `is_git` — whether it's a git repo
- `branch` — current branch name
- `sha` — short commit hash
- `dirty` — whether working tree has changes (`git status --porcelain`)
- `changed_files` — count of changed files
- `commit_msg` — last commit subject
- `commit_time` — relative time string (e.g., "2 hours ago")
- `remote` — origin URL
- `ahead` / `behind` — commits ahead/behind the tracking branch

Returns `{ projects[], root }`. Projects sorted by name (case-insensitive).

Helper `_run(args, cwd)` executes git commands safely, returning `None` on failure.

## Frontend

**File:** `apps/projects/frontend/src/App.jsx`
**Stack:** React + Vite + Tailwind

### Layout

- **Sticky header**: title "oh-projects", subtitle "git overview", root path, repo count, dirty count, live-pulse dot (15s interval), refresh button
- **Table** with columns: Repo, Branch, Last Commit (sha + message), When, Status (dirty/clean badge + sync arrows), Remote (clickable link)

### Components

All defined in `App.jsx`:

- **`ProjectRow`** — table row for one repo
- **`StatusBadge`** — colored pill: amber "dirty", green "clean", gray "no git"
- **`SyncBadge`** — shows ↑N / ↓N for ahead/behind
- **`RemoteLink`** — parses git remote URL (SSH or HTTPS) into a clickable link; shortens known hosts (github.com, gitlab.com, bitbucket.org) to just `org/repo`

### Behavior

- Auto-refreshes every 15 seconds via `useProjects(interval)` hook
- Manual refresh button in the header
- Shows "scanning…" while loading, error banner on failure

### API Module

`src/api.js` exports `api.projects()` which calls `GET /apps/projects/api/projects`.
