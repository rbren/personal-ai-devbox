# Agent Guide

This repository contains a set of prompts that, together, fully describe the **openhands-remote** project. An agent (or team of subagents) can use them to recreate the entire application from scratch on a fresh machine.

Once you're done building the system for the first time, start all the servers, including the agent-server, and check that everything is running OK.

If you expose anything to the open internet (e.g. host 0.0.0.0) be sure to at least put it behind basic auth. You can come up with a good password and give it to the user.

## Contributing Back
If you learn anything as you go about implementing these prompts--especially things the user asks you to fix!--please
contribute back to this repository (https://github.com/rbren/personal-ai-devbox) by opening a pull request with your learnings.

## Prompt Inventory

| Prompt | Scope |
|--------|-------|
| `prompts/architecture.md` | Project structure, nginx, SSL, basic auth, fail2ban, homepage shell, shared deps, Docker, data storage, port map |
| `prompts/agent-server.md` | OpenHands agent server: REST/WebSocket API, configuration, Docker images, TypeScript client, event types |
| `prompts/status.md` | System status dashboard (CPU, memory, disk, processes, ports, network) |
| `prompts/files.md` | File browser and editor |
| `prompts/projects.md` | Git project listing for `~/git/` |
| `prompts/secrets.md` | Key-value secret store (`~/.openhands/remote/secrets.json`) |
| `prompts/terminal.md` | Multi-tab browser terminal backed by real PTY processes |
| `prompts/skills.md` | Skill (SKILL.md) CRUD editor |
| `prompts/logs.md` | Log file viewer with tailing |
| `prompts/mcp.md` | MCP server configuration editor (`~/.openhands/remote/mcp.json`) |
| `prompts/conversations.md` | Chat UI for OpenHands agent-server conversations |
| `prompts/llm.md` | LLM settings page (model, API key, base URL) — reads model list from agent-server |
| `prompts/scheduled.md` | Cron-based scheduled conversation launcher |
| `prompts/sms.md` | Twilio SMS webhook that creates agent conversations |
| `prompts/hud.md` | Draggable card dashboard for monitoring live conversations |
| `prompts/kanban.md` | Kanban board view of conversations grouped by project and status |

## Execution Order

### Phase 0 — Foundation (must be first)

**`prompts/architecture.md`** + **`prompts/agent-server.md`**

`prompts/architecture.md` sets up the host: nginx config, SSL certs, basic auth, fail2ban, the homepage shell, the `apps/` directory layout, and shared tooling. Everything else depends on this.

`prompts/agent-server.md` documents the OpenHands agent server that powers all conversations. It covers the REST and WebSocket APIs, configuration, environment variables, event types, Docker images, and the TypeScript client. Any app that creates or streams conversations needs this as context.

### Phase 1 — Standalone apps (parallelizable)

These apps have no build-time or runtime dependencies on other apps. They can all be built **in parallel** by separate subagents:

- `prompts/status.md`
- `prompts/files.md`
- `prompts/projects.md`
- `prompts/secrets.md`
- `prompts/terminal.md`
- `prompts/skills.md`
- `prompts/logs.md`
- `prompts/mcp.md`
- `prompts/llm.md`

### Phase 2 — Conversations (after Phase 0)

**`prompts/conversations.md`**

Requires the `@openhands/typescript-client` package (built from `~/git/typescript-client`) which is set up in Phase 0. The HUD and Kanban apps import source files from `apps/conversations/src/`, so conversations must be built first.

This also requires the [OpenHands agent-server](https://github.com/All-Hands-AI/OpenHands) to be running on port 4004 as its backend — but that's a runtime dependency, not a build-time one.

### Phase 3 — HUD (after Phase 2)

**`prompts/hud.md`**

Imports hooks and components from `apps/conversations/src/` via the `@assistant` Vite alias, and uses `@openhands/typescript-client`. Must be built after conversations.

### Phase 4 — Kanban (after Phase 3)

**`prompts/kanban.md`**

Imports from both `apps/hud/src/` (`@hud` alias) and `apps/conversations/src/` (`@assistant` alias). Must be built after both conversations and hud.

### Phase 5 — Agent-server–dependent apps (after Phase 1, parallelizable)

These apps call the agent-server API at runtime and read shared config files. They have no build-time dependency on other apps, but should be built after Phase 1 so that `secrets.json` and `assistant-settings.json` infrastructure is in place:

- `prompts/scheduled.md`
- `prompts/sms.md`

These two can be built **in parallel**.

## Dependency Graph

```
architecture.md + agent-server.md
 ├── status.md        ──┐
 ├── files.md         ──┤
 ├── projects.md      ──┤  Phase 1 (parallel)
 ├── secrets.md       ──┤
 ├── terminal.md      ──┤
 ├── skills.md        ──┤
 ├── logs.md          ──┤
 ├── mcp.md           ──┤
 ├── llm.md           ──┘
 │
 ├── conversations.md       ← Phase 2
 │    └── hud.md            ← Phase 3
 │         └── kanban.md    ← Phase 4
 │
 └── scheduled.md  ─┐
     sms.md         ─┘      ← Phase 5 (parallel, after secrets.md exists)
```

## Hard Dependencies

These are the constraints that **must not** be violated:

1. **`prompts/architecture.md` before everything.** It creates the directory structure, nginx config, homepage shell, and shared dependencies that all apps assume exist.
2. **`prompts/conversations.md` before `prompts/hud.md`.** HUD imports React hooks and components from `apps/conversations/src/` at build time.
3. **`prompts/hud.md` before `prompts/kanban.md`.** Kanban imports from `apps/hud/src/` at build time.
4. **`@openhands/typescript-client` built before conversations, hud, or kanban.** All three apps resolve this package via a Vite alias to `~/git/typescript-client/dist/`. The architecture phase should clone and build it.

## Soft Dependencies

These are runtime-only; violating them won't break the build but will cause runtime errors until the dependency is satisfied:

- **`prompts/secrets.md`** should exist before **`prompts/sms.md`** and **`prompts/scheduled.md`** — they read `~/.openhands/remote/secrets.json` to inject secrets into new conversations.
- **`prompts/mcp.md`** should exist before **`prompts/sms.md`** and **`prompts/scheduled.md`** — they read `~/.openhands/remote/mcp.json` to include MCP config in conversations.
- **The OpenHands agent-server** must be running on port 4004 for conversations, scheduled, and sms to function at runtime.

## Tips for Subagent Delegation

- **Give each subagent one prompt file** plus `prompts/architecture.md` as context so it understands the overall conventions (port assignments, directory layout, dark theme colors, API prefix pattern, etc.). For apps that interact with the agent server (conversations, hud, kanban, scheduled, sms), also include `prompts/agent-server.md`.
- **Phase 1 apps are ideal for parallel subagents** — they're self-contained, follow the exact same frontend/backend pattern, and have zero cross-app imports.
- **Phases 2–4 must be sequential** due to the import chain. A single subagent handling all three may be simpler than coordinating handoffs.
- **After all apps are built**, a final agent should wire up `homepage/index.html` (nav items + iframe entries), update `nginx.conf` with all location blocks, and run a smoke test.
