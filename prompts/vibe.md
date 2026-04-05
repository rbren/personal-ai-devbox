# Vibe App

Chat-first interface for rapid prototyping and vibecoding. Unlike the Code app which is focused on working within existing repos, Vibe is a creative canvas: the user describes what they want, and the agent builds it as a live-rendered artifact.

## Ports

| Component | Port |
|-----------|------|
| Frontend  | 4040 |
| Backend (agent-server) | 4041 |
| Artifacts API | 4042 |

## Architecture

The Vibe app has three processes:

1. **Agent server** (port 4041) — a separate OpenHands agent-server instance with its own conversation store (`workspace/vibe-conversations`). This is independent from the main Code agent-server on port 4004.
2. **Artifacts API** (port 4042) — a small FastAPI service that lists, serves, and git-commits artifact files.
3. **Frontend** (port 4040) — React + Vite + TypeScript UI.

### nginx Routing

```
/apps/vibe/api/       → agent-server (port 4041)
/apps/vibe/artifacts/  → artifacts API (port 4042)
/apps/vibe/           → Vite dev server (port 4040)
```

The agent-server proxy includes WebSocket upgrade headers for real-time streaming.

## Artifacts

Each conversation gets a directory at `~/git/artifacts/{conversationId}` (with dashes stripped from the UUID). The agent writes files here — primarily `.html`, `.css`, `.js`, and `.md` files.

The artifact pane in the UI:
- Polls the artifacts API every 2 seconds for file changes
- Auto-selects `.html` files when they change (or the most recently modified file)
- Renders HTML files in a sandboxed iframe with a `<base>` tag so relative asset paths resolve
- Renders non-HTML files as Markdown
- Is resizable via a drag handle

When the agent finishes a run, artifacts are auto-committed to a git repo at `~/git/artifacts/`.

## Artifacts API

**File:** `apps/vibe/backend/main.py`
**Stack:** FastAPI + uvicorn

### Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/artifacts/{conv_id}` | List artifact files (name, size, mtime) |
| `GET` | `/api/artifacts/{conv_id}/{filename}` | Serve an artifact file |
| `POST` | `/api/artifacts/{conv_id}/commit` | Git add + commit the conversation's artifacts |

Conversation IDs are normalized by stripping dashes and sanitizing path traversal.

## Frontend

**Directory:** `apps/vibe/` (app root)
**Stack:** React + Vite + TypeScript
**Dependencies:** `@openhands/typescript-client`, `react-markdown`, `remark-gfm`

### Vite Config

```ts
base: '/apps/vibe/',
port: 4040,
resolve.alias: {
  '@openhands/typescript-client': '/root/git/typescript-client/dist/index.js',
  '@shared': '../shared',  // shared components and hooks
},
```

### Source Structure

```
src/
  App.tsx              — main layout: sidebar + chat + artifact pane
  App.css              — Vibe-specific theme overrides and artifact styles
  main.tsx             — React entry point
  constants.ts         — default tool list
  components/
    ArtifactPane.tsx   — right-side pane rendering artifacts (HTML iframe or Markdown)
  hooks/
    useArtifacts.ts    — polls artifact files and content
  utils/
    createConversation.ts — creates conversations with vibecoding system prompt
```

### Layout

Three-column layout:

1. **ConversationList** (left sidebar, from `@shared`) — conversation list with new chat, rename, delete, star. Collapsed by default.
2. **ChatView** (center, from `@shared`) — streams conversation events, sends messages.
3. **ArtifactPane** (right) — always visible, resizable. Shows artifact file tabs and renders the selected file.

### Theme

Dark purple theme via CSS custom properties in `App.css`:
- Background: `#16121e`, sidebar: `#1c1528`
- Accent: `#a78bfa`, user bubbles: `#7c5cfc`
- Agent bubbles: `#25203a`

### System Prompt

Each conversation gets a custom system prompt (built in `createConversation.ts`) that instructs the agent to:
- Save output as artifacts in the conversation's artifact directory
- Use plain HTML/CSS/JS only — no build tools, bundlers, or dev servers
- Use CDN links for external libraries (React, D3, Three.js, p5.js, Tailwind, etc.)
- Write CSS/JS in separate files with plain relative paths
- Write the HTML file **last** so the preview refreshes with all assets ready
- Be conversational and brief in chat — the artifact is the main output
- Bias toward action rather than asking clarifying questions

### URL Routing

- `/apps/vibe/` — no conversation selected
- `/apps/vibe/{id}` — specific conversation open

Embedded iframe behavior is the same pattern as the Code app: `postMessage` to parent for URL sync, parent relays via `vibe-navigate`/`vibe-select` messages.

### Shared Code

The Vibe app imports heavily from `apps/shared/`:
- `ConversationList`, `ChatView` components
- `useSettings`, `useConversationList`, `useAgentConversation` hooks
- `conversationSetup` utilities (skills, MCP, secrets)
- `markdownComponents` for rendering

The only Vibe-specific frontend code is the artifact pane, artifact hook, theme overrides, and conversation creation logic (which builds the vibecoding system prompt and sets the workspace to the artifacts directory).

## Default Tools

```ts
[
  { name: 'terminal' },
  { name: 'file_editor' },
  { name: 'browser_tool_set' },
  { name: 'glob' },
  { name: 'grep' },
]
```

Note: no `planning_file_editor` or `delegate` — Vibe conversations are meant to be lightweight and focused.
