# Agent Server

The OpenHands agent server is the backend that powers every conversation in this project. It's a FastAPI application from the [OpenHands Software Agent SDK](https://github.com/OpenHands/software-agent-sdk) that exposes REST and WebSocket APIs for creating, running, and streaming AI agent conversations. The conversations, HUD, kanban, scheduled, and SMS apps all talk to it.

## Source & Documentation

| Resource | Location |
|----------|----------|
| Source code | [`openhands-agent-server/`](https://github.com/OpenHands/software-agent-sdk/tree/main/openhands-agent-server/openhands/agent_server) in the `software-agent-sdk` monorepo |
| OpenAPI spec (swagger) | [`swagger-doc.json`](https://github.com/OpenHands/typescript-client/blob/main/swagger-doc.json) in the `typescript-client` repo |
| Interactive docs | `http://localhost:4004/docs` (Swagger UI) and `/redoc` (ReDoc) when the server is running |
| SDK documentation | [docs.openhands.dev/sdk](https://docs.openhands.dev/sdk) |
| API reference | [docs.openhands.dev/sdk/guides/agent-server/api-reference](https://docs.openhands.dev/sdk/guides/agent-server/api-reference/server-details/alive) |

## Running the Server

The agent server is run from a local clone of `software-agent-sdk` at `~/git/software-agent-sdk`:

```bash
cd ~/git/software-agent-sdk
OH_SESSION_API_KEYS_0=your-session-key \
OH_SECRET_KEY=$(openssl rand -hex 32) \
OH_ALLOW_CORS_ORIGINS='["*"]' \
  uv run python -m openhands.agent_server --port 4004
```

Or via Docker:

```bash
docker run -p 4004:8000 \
  -e OH_ENABLE_VNC=false \
  -e SESSION_API_KEY="$SESSION_API_KEY" \
  -e OH_ALLOW_CORS_ORIGINS='["*"]' \
  ghcr.io/openhands/agent-server:main-python
```

Command-line flags:

| Flag | Default | Description |
|------|---------|-------------|
| `--host` | `0.0.0.0` | Bind address |
| `--port` | `8000` | Bind port (we use `4004`) |
| `--reload` | off | Auto-reload on code changes (dev mode) |
| `--check-browser` | — | Verify Chromium/Playwright works, then exit |

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `OH_SESSION_API_KEYS_0` | Recommended | API key for authenticating requests. Sent as `Authorization: Bearer <key>` or `X-Session-API-Key` header. If not set, auth is disabled (open access). |
`SESSION_API_KEY` | — | Legacy alias for `OH_SESSION_API_KEYS_0` |
| `OH_SECRET_KEY` | Recommended | Encryption key for persisting LLM API keys and conversation secrets across restarts. Without it, secrets are redacted on disk and lost on restart. |
| `OH_ALLOW_CORS_ORIGINS` | — | JSON array of allowed CORS origins (localhost always allowed) |
| `OH_CONVERSATIONS_PATH` | `workspace/conversations` | Where conversation data is stored |
| `OH_ENABLE_VNC` | `false` | Enable VNC desktop inside the container |
| `OH_ENABLE_VSCODE` | `true` | Enable VSCode server on port 8001 |
| `OPENHANDS_AGENT_SERVER_CONFIG_PATH` | `workspace/openhands_agent_server_config.json` | Path to JSON config file (alternative to env vars) |

## REST API

All authenticated routes are behind `/api/`. Health and server-info routes are at the root.

### Server Details

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/alive` | Liveness probe |
| `GET` | `/health` | Health check |
| `GET` | `/server_info` | Server version and capabilities |

### Conversations

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/conversations` | Create a new conversation (accepts agent config, workspace, initial message, secrets, tools, plugins, hooks) |
| `GET` | `/api/conversations/search` | List/search conversations (paginated, filterable by status) |
| `GET` | `/api/conversations/count` | Count conversations matching filters |
| `GET` | `/api/conversations/{id}` | Get conversation details |
| `GET` | `/api/conversations?ids=` | Batch get conversations |
| `PATCH` | `/api/conversations/{id}` | Update conversation metadata (e.g. title) |
| `DELETE` | `/api/conversations/{id}` | Delete a conversation |
| `POST` | `/api/conversations/{id}/run` | Start the agent loop |
| `POST` | `/api/conversations/{id}/pause` | Pause a running conversation |
| `POST` | `/api/conversations/{id}/secrets` | Update conversation secrets |
| `POST` | `/api/conversations/{id}/confirmation_policy` | Set confirmation policy (always, risky, never) |
| `POST` | `/api/conversations/{id}/security_analyzer` | Set security analyzer |
| `POST` | `/api/conversations/{id}/generate_title` | Generate a title via LLM |
| `POST` | `/api/conversations/{id}/ask_agent` | Ask the agent a question without affecting state |
| `POST` | `/api/conversations/{id}/condense` | Force context condensation |

### Events

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/conversations/{id}/events/search` | Search/list events (paginated, filterable by kind, source, body, timestamp range) |
| `GET` | `/api/conversations/{id}/events/count` | Count events matching filters |
| `GET` | `/api/conversations/{id}/events/{event_id}` | Get a single event |
| `GET` | `/api/conversations/{id}/events?event_ids=` | Batch get events |
| `POST` | `/api/conversations/{id}/events` | Send a message (with optional `run: true` to auto-start the agent) |
| `POST` | `/api/conversations/{id}/events/respond_to_confirmation` | Accept or reject a pending action |

### Files, Bash, Git

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/file/download/{path}` | Download a file from the workspace |
| `POST` | `/api/file/upload/{path}` | Upload a file to the workspace |
| `POST` | `/api/bash/execute_bash_command` | Execute a bash command synchronously |
| `POST` | `/api/bash/start_bash_command` | Start an async bash command |
| `GET` | `/api/bash/bash_events/search` | Search bash events |
| `GET` | `/workspace` | Get workspace info |

### Other

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/tools/` | List available tools |
| `GET` | `/api/vscode/status` | VSCode server status |
| `GET` | `/api/vscode/url` | VSCode server URL |
| `GET` | `/api/desktop/url` | VNC desktop URL |
| `GET` | `/docs` | Swagger UI |
| `GET` | `/redoc` | ReDoc |

## WebSocket API

Real-time event streaming uses WebSocket connections under `/sockets/`:

### Conversation Events

```
ws[s]://host:port/sockets/events/{conversation_id}?session_api_key=<key>&resend_all=false
```

- Streams all events for a conversation in real time
- Accepts JSON messages over the socket (parsed as `Message` and auto-sent to the agent)
- Set `resend_all=true` to replay all existing events on connect
- Auth via `session_api_key` query parameter (browsers can't send custom headers on WS connections)
- Close codes: `4001` = auth failure, `4004` = conversation not found

### Bash Events

```
ws[s]://host:port/sockets/bash-events?session_api_key=<key>&resend_all=false
```

- Streams bash execution events
- Accepts `ExecuteBashRequest` messages to start new commands

## Event Types

Events are the core data model. Every action and observation in a conversation is an event.

| `kind` | Source | Description |
|--------|--------|-------------|
| `MessageEvent` | agent/user | Chat messages — the main conversation content |
| `ActionEvent` | agent | Tool call the agent is making (`tool_name`, `action`, `thought`) |
| `ObservationEvent` | environment | Result of a tool call (`tool_name`, `observation`) |
| `AgentErrorEvent` | agent | Error sent to the LLM for self-correction |
| `ConversationErrorEvent` | environment | Fatal conversation-level error |
| `UserRejectObservation` | environment | User or hook rejected an action |
| `PauseEvent` | user | Conversation paused |
| `ConversationStateUpdateEvent` | environment | State change (most useful: `key='execution_status'`) |
| `Condensation` | environment | Context window compacted |
| `CondensationSummaryEvent` | environment | Summary after condensation |
| `SystemPromptEvent` | agent | System prompt and tool list (internal) |

## Creating a Conversation

A `POST /api/conversations` request body looks like this. You should set the workspace based on
where the user keeps their code.

```json
{
  "agent": {
    "llm": {
      "model": "anthropic/claude-sonnet-4-5-20250929",
      "api_key": "sk-..."
    },
    "tools": [
      { "name": "terminal" },
      { "name": "file_editor" },
      { "name": "browser_tool_set" },
      { "name": "task_tracker" },
      { "name": "glob" },
      { "name": "grep" }
    ]
  },
  "workspace": {
    "working_dir": "/home/username/git/todo-list"
  },
  "initial_message": {
    "role": "user",
    "content": [{ "type": "text", "text": "Hello!" }]
  },
  "secrets": {
    "GITHUB_TOKEN": "ghp_..."
  },
  "confirmation_policy": { "kind": "never_confirm" },
  "max_iterations": 500
}
```

**⚠️ `working_dir` must be an absolute, writable path that exists on the host.** If it doesn't exist or isn't writable, conversation creation will fail with a 500 error (the server calls `Path(working_dir).mkdir(parents=True, exist_ok=True)` internally, and it crashes if the parent path is invalid). Use a real directory like `/tmp/workspace`.

**Tool names are lowercase snake_case.** The registered tools are:

| Name | Description |
|------|-------------|
| `terminal` | Bash/shell execution |
| `file_editor` | File read/write/patch |
| `browser_tool_set` | Web browsing (requires Chromium) |
| `task_tracker` | Task list management |
| `glob` | File pattern matching |
| `grep` | Content search |
| `apply_patch` | Apply unified diffs |

## Data Storage

Conversations are stored as files on disk:

```
workspace/conversations/
├── {conversation_id}/
│   ├── metadata.json       # Conversation config + metadata
│   └── events.jsonl        # Append-only event log
```

When `OH_SECRET_KEY` is set, LLM API keys and conversation secrets are encrypted at rest. Without it, they're redacted and lost on restart.

## TypeScript Client

The [`@openhands/typescript-client`](https://github.com/OpenHands/typescript-client) provides a browser-compatible TypeScript SDK for the agent server. It's the main way the frontend apps in this project communicate with the agent server.

Install:
```bash
npm install @openhands/typescript-client --registry=https://npm.pkg.github.com
# or use a local build:
npm install /root/git/typescript-client
```

The typescript-client repo includes a [`frontend.md` skill](https://github.com/OpenHands/typescript-client/blob/main/.agent/skills/frontend.md) that provides detailed instructions for building React chat frontends, including:
- A complete `useAgentConversation` hook
- Event rendering for every event type
- Conversation listing via `ConversationManager`
- Vite configuration for local package usage
- Authentication setup
- Tool configuration

This skill is the best reference for building or modifying the conversations, HUD, and kanban apps.

### Key TypeScript Classes

```typescript
import {
  Conversation,        // Create/resume conversations
  Agent,               // LLM config (model + api_key)
  Workspace,           // Points at the agent server host
  ConversationManager, // List/search conversations
} from '@openhands/typescript-client';

// Create a conversation
const agent = new Agent({
  llm: { model, api_key: apiKey },
  tools: [
    { name: 'terminal' },
    { name: 'file_editor' },
    { name: 'browser_tool_set' },
    { name: 'glob' },
    { name: 'grep' },
  ],
});
const workspace = new Workspace({
  host: 'http://localhost:4004',
  workingDir: '/tmp/workspace',
  apiKey: sessionKey,
});
const conv = new Conversation(agent, workspace, { callback: onEvent });
await conv.start({ initialMessage: 'Hello!' });
await conv.startWebSocketClient();
await conv.run();
```

### Reconnecting to an Existing Conversation

To reconnect to a conversation that already exists (e.g. after a page reload), pass the `conversationId` and omit the `initialMessage`:

```typescript
const conv = new Conversation(agent, workspace, {
  conversationId,       // existing conversation ID
  callback: onEvent,    // live event handler
});
await conv.start();                          // connect without initial message
const history = await conv.state.events.getEvents();
const status = await conv.state.getAgentStatus();
await conv.startWebSocketClient();           // subscribe to live events

// Cleanup when unmounting:
conv.stopWebSocketClient()
```

## Relationship to This Project

In this project's architecture:

- The agent server runs on **port 4004** (see `prompts/architecture.md`)
- nginx proxies `/apps/conversations/api/` → `http://127.0.0.1:4004/` (note: no `/api/` prefix stripping — the agent server's own route structure is used directly)
- WebSocket upgrade headers are passed through for real-time streaming
- The **conversations** app is the primary frontend — it uses `@openhands/typescript-client` to create and stream conversations
- The **HUD** and **kanban** apps import hooks from the conversations app and also use the typescript-client
- The **scheduled** and **SMS** apps create conversations by POSTing directly to the agent server's REST API from their Python backends
- Configuration for the agent server connection is stored in `~/.openhands/remote/assistant-settings.json`

## Webhooks

The agent server supports webhooks for event notifications. Configure via the JSON config file or environment:

```json
{
  "webhooks": [{
    "base_url": "https://your-endpoint.com",
    "event_buffer_size": 5,
    "flush_delay": 30.0,
    "num_retries": 3,
    "retry_delay": 5,
    "headers": { "Authorization": "Bearer token" }
  }]
}
```

Events are buffered and sent in batches to `{base_url}/events`. Conversation info is sent to `{base_url}/conversations`.

## Docker Images

The agent server publishes Docker images to `ghcr.io/openhands/agent-server`:

| Tag suffix | Contents |
|------------|----------|
| `-python` (source) | Full source + venv — good for dev and debugging |
| (binary) | PyInstaller binary — smaller, for production |
| `-minimal` variants | No VNC, no VSCode, no desktop — headless only |

Full images include: Docker, VSCode Web (port 8001), VNC desktop, Chromium, GitHub CLI, tmux.

## Chromium

The agent server's browser tool requires Chromium via Playwright. You MUST install this.

## Chromium Requirement

The agent server requires Chromium for browser tool operations. Without it, the
server will throw an exception when initializing the browser tool:

```
Exception: Chromium is required for browser operations but is not installed.
```

### Installation

Install Chromium via Playwright (recommended):

```bash
uvx playwright install chromium --no-shell
```

To also install system-level dependencies (requires sudo):

```bash
uvx playwright install chromium --with-deps --no-shell
```

The Chromium binary is installed to `~/.cache/ms-playwright/chromium-*/`.

### Troubleshooting

If `--with-deps` fails due to lack of sudo access, install Chromium without
system deps first (`--no-shell` only), then manually install any missing shared
libraries reported at runtime. Common missing libraries on Ubuntu/Debian can be
installed with:

```bash
sudo apt-get install -y libnss3 libatk1.0-0 libatk-bridge2.0-0 \
  libcups2 libdrm2 libxkbcommon0 libxcomposite1 libxdamage1 \
  libxfixes3 libxrandr2 libgbm1 libpango-1.0-0 libcairo2 libasound2
