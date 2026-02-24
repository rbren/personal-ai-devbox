# Conversations App


![conversations screenshot](conversations.png)
Chat UI for interacting with OpenHands agent conversations. This is the primary app — it creates, lists, and streams conversations running on the OpenHands agent server.

## Ports

| Component | Port |
|-----------|------|
| Frontend  | 4003 |
| Backend   | 4004 (external agent-server) |

## Backend

The conversations app does **not** have its own backend. It relies on the [OpenHands agent-server](https://github.com/All-Hands-AI/OpenHands) from the `software-agent-sdk` project, running on port 4004.

### Starting the Agent Server

```bash
cd ~/git/software-agent-sdk
OH_SESSION_API_KEYS_0=your-session-key OH_ALLOW_CORS_ORIGINS='["*"]' uv run agent-server --port 4004
```

- `OH_SESSION_API_KEYS_0` — session API key for authentication
- `OH_ALLOW_CORS_ORIGINS` — CORS whitelist

### nginx Proxy

The conversations API proxy is special — it proxies `/apps/conversations/api/` to `http://127.0.0.1:4004/` (not `/api/`), because the agent-server has its own route structure. WebSocket upgrade headers are included for streaming.

### Chromium

The agent server includes a browser tool backed by Playwright. Chromium must be available (installed via `uvx playwright install chromium --with-deps --no-shell`). When running as root, the server automatically adds `--no-sandbox`.

## Frontend

**Directory:** `apps/conversations/` (app root, not in a `frontend/` subdirectory)
**Stack:** React + Vite + TypeScript + Tailwind
**Dependencies:** `@openhands/typescript-client` (local build from `~/git/typescript-client`)

### Vite Config

```ts
base: '/apps/conversations/',
port: 4003,
resolve.alias: {
  '@openhands/typescript-client': '/root/git/typescript-client/dist/index.js',
},
optimizeDeps.exclude: ['@openhands/typescript-client'],
```

The local typescript-client is excluded from pre-bundling so Vite uses the pre-built dist as-is.

### Source Structure

```
src/
  App.tsx              — main layout: sidebar + chat + plan pane
  App.css              — styles
  main.tsx             — React entry point
  constants.ts         — shared constants
  components/
    ChatView.tsx       — message stream and input
    ConversationList.tsx — sidebar conversation list
    EventBubble.tsx    — renders individual conversation events
    PlanPane.tsx       — right-side pane showing agent's plan file
  hooks/
    useAgentConversation.ts — streams events from a conversation
    useConversationList.ts  — fetches/manages conversation list
    useProjects.ts          — fetches git projects for working dir picker
    useSettings.ts          — reads/writes assistant settings (from localStorage, synced with LLM app)
  utils/
    createConversation.ts   — helper to POST a new conversation
```

### App.tsx Layout

Three-column layout:

1. **ConversationList** (left sidebar): searchable conversation list with new chat button, refresh, delete, rename. Collapsible via toggle.
2. **ChatView** (center): streams conversation events in real-time, shows input box for sending messages
3. **PlanPane** (right): displays the contents of `PLAN.md` from the conversation's working directory

### URL Routing

The app supports deep-linking to conversations:

- `/apps/conversations/` — no conversation selected
- `/apps/conversations/{id}` — specific conversation open

When **embedded** in the homepage shell (iframe):
- URL changes are communicated to the parent via `postMessage({ type: 'conversations-navigate', path })`.
- The parent pushes to browser history and relays back/forward via `postMessage({ type: 'conversations-select', id })`.

When **standalone**:
- Uses `history.pushState()` / `popstate` directly.

### Settings

LLM settings (model, API key, base URL, session key) are configured in the separate **LLM app** (`apps/llm/`, see `prompts/llm.md`), not in the conversations app. The conversations app no longer has its own settings modal.

The `useSettings` hook reads settings from `localStorage` (key `openhands-chat-settings`) and listens for `storage` events so changes made in the LLM app propagate immediately across iframes without a page reload. Settings are passed as context to conversation creation.

The underlying data lives in `~/.openhands/remote/assistant-settings.json`:

```json
{
  "agentServerUrl": "http://localhost:4004",
  "model": "claude-sonnet-4-6",
  "apiKey": "",
  "sessionKey": ""
}
```


### Creating Conversations

`createConversation.ts` POSTs to the agent server with:
- `agent.llm.model` and `agent.llm.api_key` from settings
- `agent.tools` — default tool list
- `agent.mcp_config` — loaded from the MCP app's config endpoint
- `workspace.working_dir` — selected project directory
- `initial_message` — user's first message with `run: true`

After creation, it generates a title, patches the conversation, and injects secrets from the secrets API.

### Shared Code

The `src/hooks/useSettings.ts` and `src/hooks/useProjects.ts` hooks are imported by the HUD and Kanban apps via the `@assistant` Vite alias. This is a build-time dependency.
