# Logs App


![logs screenshot](logs.png)
Log file viewer with live tailing. Shows all `*.log` files from the project's `logs/` directory, grouped by app name in a sidebar.

## Ports

| Component | Port |
|-----------|------|
| Frontend  | 4017 |
| Backend   | 4018 |

## Backend

**File:** `apps/logs/backend/main.py`
**Stack:** FastAPI
**Dependencies:** `fastapi`, `uvicorn[standard]`

### Log Directory

Default: `<project-root>/logs/` (resolved from `__file__`). Can be overridden with `LOG_DIR` env var.

The log directory contains files like `status-backend.log`, `files-frontend.log`, etc., written by each app's `start.sh`.

### Endpoints

#### `GET /api/logs`

Lists all `*.log` files in the log directory. Returns `{ files: [{ name, size, modified }] }`. Sorted alphabetically.

#### `GET /api/logs/{name}?offset=0`

Reads a log file starting from byte `offset`. Validates that:
- Name ends with `.log`
- Name contains no `/`, `\`, or `..` (path traversal protection)

Returns `{ content, size }`. Content is decoded as UTF-8 with `errors="replace"`. The `size` field is the total file size (used by the frontend as the next offset).

## Frontend

**File:** `apps/logs/frontend/src/App.jsx`
**Stack:** React + Vite + Tailwind

### Layout

- **Left sidebar** (192px): grouped file list. Files are grouped by app name (prefix before the first `-`). Each group has a header, and each file shows just the suffix (e.g., "backend", "frontend").
- **Main area**: toolbar + scrollable log output

### Tailing Behavior

1. Frontend polls `GET /api/logs/{name}?offset=N` every **1 second** (`POLL_MS = 1000`)
2. Maintains a running byte offset — each response returns `size`, which becomes the next request's `offset`
3. Incoming content is split by `\n`. An incomplete trailing line is buffered in `bufRef` and prepended to the next chunk.
4. Lines are capped at `LINE_LIMIT = 5000` — older lines are dropped when the limit is reached.

### File List Polling

The file list re-fetches every **5 seconds** (`FILE_POLL_MS = 5000`) to pick up new or removed log files.

### Auto-Scroll

- Enabled by default
- Automatically scrolls to the bottom when new lines arrive
- Disables when the user scrolls up (more than 60px from the bottom)
- Re-enables when the user scrolls back to the bottom
- Toggle checkbox in the toolbar

### Line Coloring

Lines are syntax-highlighted with `lineClass()`:
- **Red** (`text-red-400`): lines containing "error" (case-insensitive)
- **Yellow** (`text-yellow-400`): lines containing "warn" or "warning"
- **Orange** (`text-orange-400`): lines containing HTTP 4xx/5xx status codes (` 4xx ` or ` 5xx `)
- **Green** (`text-emerald-400`): lines containing "ready", "success", "compiled", "✓", or "done"
- **Default** (`text-[#c9d1d9]`): all other lines

### Toolbar

- Shows the selected filename in monospace
- Auto-scroll checkbox
- "clear" button (resets lines and offset, starts fresh)
