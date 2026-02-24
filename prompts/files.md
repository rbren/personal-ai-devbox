# Files App


![files screenshot](files.png)
File browser and text editor for the server's filesystem. Two-pane layout: directory tree on the left, code editor on the right.

## Ports

| Component | Port |
|-----------|------|
| Frontend  | 4005 |
| Backend   | 4006 |

## Backend

**File:** `apps/files/backend/main.py`
**Stack:** FastAPI
**Dependencies:** `fastapi`, `uvicorn[standard]`

### Endpoints

#### `GET /api/ls?path=/`

Lists a directory. Returns `{ path, parent, entries[] }`. Each entry: `name`, `path`, `is_dir`, `size`, `modified` (mtime), `mode` (octal). Entries sorted: directories first, then case-insensitive alphabetical. Returns 404 if path doesn't exist, 400 if not a directory, 403 on permission error.

#### `GET /api/read?path=...`

Reads a file's text content. Returns `{ path, content }`. Uses `errors="replace"` for non-UTF-8 bytes. Returns 400 if path is not a file.

#### `POST /api/write`

Body: `{ path, content }`. Writes text content to a file, creating parent directories as needed. Returns `{ ok, path }`.

#### `POST /api/create`

Body: `{ path, is_dir }`. Creates a new empty file or directory. Returns 409 if path already exists. Creates parent directories for files.

#### `POST /api/rename`

Body: `{ src, dst }`. Renames/moves a file or directory. Returns 409 if destination exists.

#### `POST /api/delete`

Body: `{ path }`. Deletes a file or directory (recursive via `shutil.rmtree` for directories).

## Frontend

**File:** `apps/files/frontend/src/App.jsx`
**Stack:** React + Vite + Tailwind

### Layout

- **Left sidebar** (256px fixed): breadcrumb navigation, optional path input bar, file tree, status messages
- **Right pane** (flex): file path header + text editor

### Components

Files in `src/components/`:

- **`Breadcrumb`** — clickable path segments for quick directory navigation
- **`FileTree`** — list of entries with icons (📁/📄), click to navigate directories or select files, right-click for context menu, inline rename support
- **`ContextMenu`** — right-click menu with Rename and Delete options
- **`Editor`** — text editor pane; fetches file content via `/api/read`, saves via `/api/write`

### Modals

Defined inline in `App.jsx`:

- **`CreateModal`** — new file or folder dialog with File/Folder toggle, name input, and path display
- **`DeleteModal`** — confirmation dialog showing the entry name and type

### Behavior

- Clicking a directory navigates into it; clicking a file opens it in the editor
- Right-click any entry to rename or delete
- Press Escape to close context menus and cancel renames
- The `+` button in the sidebar header opens the create modal
- A `PathInput` component allows typing an arbitrary path to navigate to (activated programmatically)
- Status messages (flash notifications) appear at the bottom of the sidebar and auto-dismiss after 3 seconds

### API Module

`src/api.js` exports an `api` object with methods: `ls(path)`, `read(path)`, `write(path, content)`, `create(path, isDir)`, `rename(src, dst)`, `delete(path)`. All call `/apps/files/api/...`.
