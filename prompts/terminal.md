# Terminal App


![terminal screenshot](terminal.png)
Multi-tab browser terminal backed by real PTY (pseudo-terminal) processes. Each tab is a full bash login shell.

## Ports

| Component | Port |
|-----------|------|
| Frontend  | 4013 |
| Backend   | 4014 |

## Backend

**File:** `apps/terminal/backend/main.py`
**Stack:** FastAPI + Python pty/os/asyncio
**Dependencies:** `fastapi`, `uvicorn[standard]`

### Architecture

The backend manages a dict of `TerminalSession` objects keyed by short UUIDs. Each session holds:

- `master_fd` — the master side of the PTY
- `pid` — the child bash process ID
- `_queues` — a set of `asyncio.Queue` objects for fan-out to multiple WebSocket clients

### Endpoints

#### `POST /api/terminals`

Creates a new terminal session:

1. Opens a PTY pair via `pty.openpty()`
2. Forks a child process
3. Child: calls `os.setsid()`, sets the slave as the controlling terminal via `TIOCSCTTY`, dups stdin/stdout/stderr to the slave fd, closes all other fds (3–4096), then `execvpe(shell, [shell, "-l"], env)` with `TERM=xterm-256color` and `COLORTERM=truecolor`
4. Parent: closes slave fd, registers the master fd with `loop.add_reader()` for async non-blocking reads, stores the session

Returns `{ id }` (8-char UUID prefix).

#### `DELETE /api/terminals/{id}`

Sends SIGTERM to the child, closes the master fd, removes the event loop reader, and signals all subscriber queues with `None` (EOF).

#### `GET /api/terminals`

Returns `{ terminals: [id, ...] }` — list of active session IDs.

#### `WS /api/terminals/{id}/ws`

WebSocket endpoint for bidirectional terminal I/O:

- **PTY → client**: `_on_output()` reads up to 4096 bytes from master_fd, pushes to all subscriber queues. A background task awaits each queue and sends binary frames to the WebSocket.
- **Client → PTY**: receives JSON text frames:
  - `{ type: "input", data: "..." }` — writes UTF-8 encoded data to master_fd
  - `{ type: "resize", rows: N, cols: N }` — sends `TIOCSWINSZ` ioctl to resize the PTY

Two asyncio tasks run concurrently (`pty_to_ws` and `ws_to_pty`). When either task ends (disconnect or EOF), both are cancelled and the queue is unsubscribed.

### nginx Configuration

The terminal API location block has special settings:

```nginx
proxy_read_timeout 3600s;  # keep WebSocket alive for up to 1 hour
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
```

## Frontend

**File:** `apps/terminal/frontend/src/App.jsx`
**Stack:** React + Vite + xterm.js
**Dependencies:** `@xterm/xterm`, `@xterm/addon-fit`

### Layout

- **Tab bar** at the top: one tab per terminal, each with a label and close (×) button. A `+` button creates new terminals.
- **Terminal panes** fill the remaining space. All panes are mounted simultaneously in the DOM but only the active one is visible (`display: block` vs `display: none`). This keeps bash processes alive during tab switches.

### `TerminalPane` Component

`src/components/TerminalPane.jsx` — manages a single xterm.js instance:

1. On mount: creates an `xterm.Terminal` with `fontFamily: 'Menlo, Monaco, monospace'`, `fontSize: 14`, dark theme colors
2. Opens a WebSocket to `/apps/terminal/api/terminals/{id}/ws`
3. Pipes incoming binary data from WebSocket to `terminal.write()`
4. Pipes terminal input (`terminal.onData`) to WebSocket as `{ type: "input", data }` JSON
5. Uses `FitAddon` + `ResizeObserver` to auto-size the terminal to its container
6. On resize, sends `{ type: "resize", rows, cols }` to the WebSocket
7. Re-fits when the tab becomes active (visibility changes from hidden to visible)

### Behavior

- Auto-opens one terminal on first load
- Tab counter increments for labels: "Terminal 1", "Terminal 2", etc.
- Closing a tab sends `DELETE /api/terminals/{id}` and switches focus to the last remaining tab
- Empty state shows a `$_` prompt and instruction to press `+`

### start.sh

The terminal start script kills stale processes on ports 4013/4014 before starting, to handle unclean shutdowns.
