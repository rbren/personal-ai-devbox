# MCP App


![mcp screenshot](mcp.png)
Configuration editor for Model Context Protocol (MCP) servers. Stores the config in `~/.openhands/remote/mcp.json`, which is read by the conversations, SMS, and scheduled apps when creating new agent conversations.

## Ports

| Component | Port |
|-----------|------|
| Frontend  | 4022 |
| Backend   | 4023 |

## Backend

**File:** `apps/mcp/backend/main.py`
**Stack:** FastAPI
**Dependencies:** `fastapi`, `uvicorn[standard]`

### Storage

File: `~/.openhands/remote/mcp.json`
Format:

```json
{
  "mcpServers": {
    "server-name": {
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-fetch"],
      "env": { "API_KEY": "..." }
    }
  }
}
```

This structure is passed directly as `mcp_config` in the agent conversation payload.

### Endpoints

#### `GET /api/config`

Returns the full `mcp.json` dict (used by the conversations app to include in conversation payloads).

#### `GET /api/servers`

Returns a flat list of `ServerEntry` objects: `[{ name, command, args[], env{} }]`. Used by the frontend.

#### `POST /api/servers` (201)

Body: `{ name, command, args[], env{} }`. Upserts (adds or updates) a server entry. Name and command must be non-empty. The `env` field is only stored if non-empty.

#### `DELETE /api/servers/{name}`

Removes a server by name. Returns 404 if not found.

## Frontend

**File:** `apps/mcp/frontend/src/App.jsx`
**Stack:** React + Vite + Tailwind

### Layout

- **Sticky header**: title "oh-mcp", subtitle "Model Context Protocol", "+ Add Server" button
- **Installed Servers section**: 3-column grid of `ServerCard` components
- **Catalog section**: 3-column grid of `CatalogCard` components for well-known servers

### Components

#### `ServerCard`

Shows server name, command preview (command + args joined), env variable keys as masked pills (`KEY=•••`). Edit (pencil) and remove (trash) buttons.

#### `CatalogCard`

Shows name, description, command preview. "Install" button (disabled and labeled "Installed" if already configured).

#### `ServerModal`

Modal form for adding or editing a server:
- **Name** (disabled when editing)
- **Command** (e.g., `uvx` or `npx`)
- **Arguments** (textarea, one per line)
- **Environment Variables** — dynamic key-value editor (`EnvEditor` component)

#### `EnvEditor`

Same pattern as the skills MetadataEditor: rows of key/value inputs with add/remove buttons. Keys are styled `uppercase`.

### Catalog

10 well-known MCP servers are hardcoded in `CATALOG`:

| Name | Command | Description |
|------|---------|-------------|
| fetch | `uvx mcp-server-fetch` | Fetch web pages |
| filesystem | `npx @modelcontextprotocol/server-filesystem /workspace` | Read/write files |
| memory | `npx @modelcontextprotocol/server-memory` | Persistent knowledge graph |
| sequential-thinking | `npx @modelcontextprotocol/server-sequential-thinking` | Structured reasoning |
| repomix | `npx repomix@1.4.2 --mcp` | Pack repos for AI |
| github | `npx @modelcontextprotocol/server-github` | GitHub API (needs `GITHUB_TOKEN`) |
| brave-search | `npx @modelcontextprotocol/server-brave-search` | Brave Search (needs `BRAVE_API_KEY`) |
| slack | `npx @modelcontextprotocol/server-slack` | Slack API (needs `SLACK_BOT_TOKEN`, `SLACK_TEAM_ID`) |
| postgres | `npx @modelcontextprotocol/server-postgres` | PostgreSQL queries |
| puppeteer | `npx @modelcontextprotocol/server-puppeteer` | Headless browser automation |

Installing a catalog entry pre-fills the modal with the catalog's command, args, and env template.

### Behavior

- Install from catalog: opens the modal pre-filled, user can customize env vars, then save
- Custom add: opens blank modal
- Edit: opens modal with current server data (name disabled)
- Delete: requires `confirm()` dialog
- Footer note: "Configured servers are passed as mcp_config when the assistant starts a new conversation."
