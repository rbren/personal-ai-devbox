# LLM Settings App

Dedicated settings page for configuring which LLM provider, model, and API key the agent uses. Extracted from the conversations app's former `SettingsModal` into its own micro-app so it appears as a standalone page in the Settings nav section.

## Ports

| Component | Port |
|-----------|------|
| Frontend  | 4026 |
| Backend   | 4027 |

## Backend

**File:** `apps/llm/backend/main.py`
**Stack:** FastAPI
**Dependencies:** `fastapi`, `uvicorn[standard]`, `httpx`

### Storage

File: `~/.openhands/remote/assistant-settings.json`

The backend reads and writes LLM-related fields (`model`, `apiKey`, `baseUrl`) into the shared `assistant-settings.json` file. Writes are **merged** with existing fields (e.g. `agentServerUrl`, `tools`, `sessionKey`) so other apps' settings are preserved.

### Endpoints

#### `GET /api/settings`

Returns `{ model, apiKey, baseUrl, sessionKey }` read from disk (with defaults for missing fields).

#### `PUT /api/settings`

Body: `{ model, apiKey, baseUrl }`. Merges into `assistant-settings.json`. Returns `{ ok: true }`.

#### `GET /api/models?verified=true|false`

Proxies the model list from the OpenHands agent-server (`http://localhost:4004`). The backend reads the session key from disk settings and passes it as `X-Session-API-Key` to the agent-server. Returns `{ models: { provider: [model, ...], ... } }`.

**Important:** This endpoint exists as a backend proxy (rather than having the frontend call the agent-server directly) because the agent-server returns `401 Unauthorized` when the session key is missing or wrong. If that 401 reaches the browser, it invalidates cached nginx basic-auth credentials and triggers a re-authentication popup. The backend converts 401s and connection errors into `{ models: {}, error: "..." }` — always returning HTTP 200 with a clean JSON body.

## Frontend

**File:** `apps/llm/frontend/src/App.jsx`
**Stack:** React + Vite + Tailwind

### Layout

- **Sticky header**: title "LLM Settings", save status indicator
- **Model card**: provider dropdown, model dropdown, "Verified only" toggle
- **Authentication card**: LLM API key (password field), Session API key (password field)
- **Advanced section**: collapsible, contains Base URL override field
- **Save button**: persists to both the backend (disk) and `localStorage`

### Behavior

- **On mount**: loads settings from `GET /api/settings`, then loads models from `GET /api/models`. Models are not fetched until settings finish loading (prevents race condition with missing session key).
- **Provider/Model dropdowns**: models are grouped by provider. Changing provider auto-selects the first model. The saved model is resolved to its provider when the model list loads.
- **Verified only toggle**: switches between the agent-server's verified model list and the full list. Reloads models on toggle.
- **Advanced toggle**: reveals the Base URL field (auto-expanded if a base URL is already saved).
- **Save**: `PUT /api/settings` to persist to disk, then writes `{ model, apiKey, sessionKey }` to `localStorage` key `openhands-chat-settings` so the conversations app picks up changes immediately.
- **Error display**: if model loading fails (e.g. bad session key), shows the error inline above the dropdowns.

### Cross-App Settings Sync

The LLM app and conversations app share settings via two mechanisms:

1. **Disk** (`~/.openhands/remote/assistant-settings.json`): the LLM backend writes here, the conversations `useSettings` hook reads from here via `GET /apps/llm/api/settings`.
2. **localStorage** (key `openhands-chat-settings`): the LLM frontend writes here on save. The conversations app's `useSettings` hook listens for `storage` events so changes propagate immediately across iframes without a page reload.

### API

All calls go to `/apps/llm/api/...` (proxied by nginx to port 4027):
- `fetch('/apps/llm/api/settings')` — load settings
- `fetch('/apps/llm/api/settings', { method: 'PUT', body })` — save settings
- `fetch('/apps/llm/api/models?verified=true')` — load model list
