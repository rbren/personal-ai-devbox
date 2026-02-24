# Scheduled App


![scheduled screenshot](scheduled.png)
Cron-based scheduler for launching OpenHands agent conversations on a timer. Uses the system crontab to trigger a shell script that POSTs new conversations to the agent server.

## Ports

| Component | Port |
|-----------|------|
| Frontend  | 4011 |
| Backend   | 4012 |

## Backend

**File:** `apps/scheduled/backend/main.py`
**Stack:** FastAPI
**Dependencies:** `fastapi`, `uvicorn[standard]`

### How It Works

1. The backend manages entries in the system crontab (via `crontab -l` and `crontab -`)
2. Each scheduled entry is a crontab line of the form:
   ```
   <5-field-cron> /path/to/post-conversation.sh "<prompt-file>" "<title>"
   ```
3. When cron fires, the shell script reads settings from `~/.openhands/remote/assistant-settings.json` and POSTs a new conversation to the agent server

### Prompt Storage

Prompts are stored as individual text files in `~/.openhands/remote/scheduled-prompts/`, with UUID filenames (e.g., `a1b2c3d4-....txt`). The crontab line references the prompt file path. This avoids shell escaping issues with long prompt text in crontab.

### Endpoints

#### `GET /api/schedule`

Reads `crontab -l`, filters for lines containing `post-conversation.sh`, and parses each into:
- `line_index` — position in the crontab (used for update/delete)
- `cron` — the 5-field cron expression
- `prompt` — the prompt text (read from the referenced file)
- `prompt_file` — path to the prompt text file
- `title` — the conversation title

Returns `{ entries[], script_path }`.

#### `POST /api/schedule` (201)

Body: `{ cron, prompt, title }`. Validates: cron must have exactly 5 fields, title must be non-empty. Writes the prompt to a new UUID file in `scheduled-prompts/`, appends a new crontab line.

#### `PUT /api/schedule/{line_index}`

Body: `{ cron, prompt, title }`. Updates an existing crontab entry. Reuses the existing prompt file if one exists, otherwise creates a new one.

#### `DELETE /api/schedule/{line_index}`

Removes the crontab line and deletes the associated prompt file. Only operates on lines containing `post-conversation.sh`.

#### `POST /api/schedule/{line_index}/run`

Fires the scheduled script immediately via `subprocess.Popen`. Does not wait for completion.

#### `GET /api/settings`

Returns the assistant settings from `~/.openhands/remote/assistant-settings.json`. Defaults:
```json
{
  "agentServerUrl": "http://localhost:4004",
  "model": "claude-sonnet-4-6",
  "apiKey": "",
  "sessionKey": ""
}
```

#### `PUT /api/settings`

Body: `{ agentServerUrl, model, apiKey, sessionKey }`. Writes the settings file.

### post-conversation.sh

**File:** `apps/scheduled/post-conversation.sh`

Shell script that:
1. Takes two arguments: `<prompt-text-or-file>` and `<title>`
2. If the first arg is a file path, reads the prompt from the file; otherwise uses it as inline text
3. Reads settings from `~/.openhands/remote/assistant-settings.json`
4. Contains an embedded Python script that:
   - POSTs to `{agentServerUrl}/api/conversations` with model, tools, MCP config, and the prompt as `initial_message` with `run: true`
   - PATCHes the conversation title
   - POSTs secrets from `~/.openhands/remote/secrets.json` to the conversation
5. Prints `Created conversation: {id} ({title})`

## Frontend

**File:** `apps/scheduled/frontend/src/App.jsx`
**Stack:** React + Vite + Tailwind

### Layout

- **Sticky header**: title "oh-scheduled", settings gear button, "+ Add" button
- **Schedule table**: rows for each entry showing cron expression, human-readable description, title, prompt (truncated), and action buttons (edit, run, delete)
- **Add form**: inline form at the bottom for creating new entries (cron, title, prompt fields)
- **Settings modal**: configure agent server URL, model, API key, session key

### Components

Defined in `App.jsx`:

- **`EditModal`** — modal for editing an existing entry's cron, title, and prompt
- **`SettingsModal`** — modal for editing assistant settings
- **`describeCron(expr)`** — converts cron expressions to human-readable text (e.g., `"0 9 * * *"` → `"daily at 09:00"`, `"*/5 * * * *"` → `"every 5 minutes"`)

### Behavior

- Schedule list auto-loads on mount
- "Run now" (play icon) fires `POST /api/schedule/{line_index}/run` and shows a flash message
- Edit opens a modal; save calls `PUT /api/schedule/{line_index}`
- Delete calls `DELETE /api/schedule/{line_index}` (no confirmation dialog)
- Settings are saved via `PUT /api/settings`
- Add form validates cron (5 fields), title (non-empty), and prompt (non-empty) before submitting
