# Status App


![status screenshot](status.png)
System monitoring dashboard showing live server metrics. Displays CPU, memory, disk, processes, open ports, and network interfaces.

## Ports

| Component | Port |
|-----------|------|
| Frontend  | 4001 |
| Backend   | 4002 |

## Backend

**File:** `apps/status/backend/main.py`
**Stack:** FastAPI + psutil
**Dependencies:** `fastapi`, `uvicorn[standard]`, `psutil`

### Endpoints

All routes are read-only (`GET`).

#### `GET /api/system`

Returns hostname, OS info, distro (parsed from `/etc/os-release`), architecture, Python version, boot time, uptime string, and:

- **cpu**: percent, logical/physical count, frequency, per-core utilization, load average
- **memory**: total/used/available in GB, percent, cached/buffers in MB
- **swap**: total/used in GB, percent

#### `GET /api/disk`

Returns `partitions[]` — each partition's device, mountpoint, fstype, total/used/free in GB, percent. Uses `psutil.disk_partitions(all=False)` and `psutil.disk_usage()`.

#### `GET /api/processes`

Query params: `limit` (default 20), `sort` ("cpu" or "mem").

Returns top processes sorted by CPU or memory: pid, name, user, status, cpu_percent, mem_percent, mem_rss_mb, threads, cmd (truncated to 80 chars).

#### `GET /api/ports`

Returns open network connections (`LISTEN` + `ESTABLISHED`). Each entry: port, ip, status, proto (TCP/UDP), pid, process name. Deduplicated by `(port, status, type)`.

#### `GET /api/network`

Returns network interfaces with IPv4 address, bytes sent/received in MB, packet counts.

## Frontend

**File:** `apps/status/frontend/src/App.jsx`
**Stack:** React + Vite + Tailwind

### Layout

- **Sticky header** with "oh-status" title, "system monitor" subtitle, and a live-pulse dot (toggles green/gray every 1s)
- **Two-column layout** (responsive):
  - Left (1/4 width): SystemInfo, CpuPanel, DiskPanel, NetworkPanel
  - Right (3/4 width): ProcessPanel, PortsPanel

### Components

Broken into separate component files in `src/components/`:

- `SystemInfo` — hostname, OS, uptime, memory bar
- `CpuPanel` — overall CPU %, per-core bars, load average
- `DiskPanel` — partition usage bars
- `ProcessPanel` — sortable process table (CPU/mem columns)
- `PortsPanel` — open connections table
- `NetworkPanel` — interface stats

All panels auto-refresh on a 2-second interval.

### Styling

Standard dark theme: bg `#0f1117`, borders `#2a2d3e`, muted text `#4a5070`, accent `#4a9eff`. Uses Tailwind utility classes directly in JSX.
