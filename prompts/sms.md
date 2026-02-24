# SMS App


![sms screenshot](sms.png)
Twilio webhook that creates an OpenHands conversation from an incoming SMS message. Also provides a web UI showing a log of received messages.

## Ports

| Component | Port |
|-----------|------|
| Frontend  | 4019 |
| Backend   | 4020 |

## Backend

**File:** `apps/sms/backend/main.py`
**Stack:** FastAPI + Twilio SDK
**Dependencies:** `fastapi`, `uvicorn[standard]`, `twilio`, `python-multipart`

### Configuration

- **`TWILIO_AUTH_TOKEN`** — loaded from `~/.openhands/remote/secrets.json` (key `TWILIO_AUTH_TOKEN`) or from the environment variable. Required at startup — the app raises `RuntimeError` if missing.
- **`ALLOWED_FROM`** — **Security-critical.** Only phone numbers on this allowlist can trigger conversation creation. This must be enforced: the webhook is exposed without HTTP basic auth (so Twilio can reach it), which means anyone who discovers the URL could create agent conversations by spoofing SMS requests. Twilio signature validation is the first line of defense, and the `ALLOWED_FROM` check is the second. Never set this to accept all senders.
- **`WEBHOOK_URL`** — `https://your-domain.example.com/apps/sms/api/webhook` (used for Twilio signature validation)

### Data Files

- **Message log:** `~/.openhands/remote/sms-messages.json` — array of message objects, max 200 entries (newest first)
- **Settings:** `~/.openhands/remote/assistant-settings.json` — shared with Scheduled and Conversations apps
- **Secrets:** `~/.openhands/remote/secrets.json` — injected into conversations
- **MCP config:** `~/.openhands/remote/mcp.json` — included in conversation payloads

### Endpoints

#### `GET /api/messages`

Returns `{ messages: [{ from, body, timestamp, conversation_id }] }`. Messages sorted newest-first.

#### `POST /api/webhook`

Twilio webhook endpoint for incoming SMS. Process:

1. Parse form data from Twilio POST
2. Validate the request signature using `twilio.request_validator.RequestValidator` against the `WEBHOOK_URL` and `TWILIO_AUTH_TOKEN`. Returns 403 if invalid.
3. Start a new conversation by calling the agent server:
   - Read settings from `assistant-settings.json`
   - POST to `{agentServerUrl}/api/conversations` with model, tools, MCP config, and the SMS body as `initial_message`
   - Generate and set a title
   - Inject secrets
4. Log the message to `sms-messages.json` (prepend, cap at 200)
5. Reply with TwiML: sends "hello world!" back if the sender is `ALLOWED_FROM`

### nginx Configuration

The SMS webhook has `auth_basic off` so Twilio can reach it without HTTP basic auth credentials:

```nginx
location /apps/sms/api/ {
    auth_basic off;
    proxy_pass http://127.0.0.1:4020/api/;
}
```

### Default Tools

When creating a conversation, the following tools are included:
- `terminal`
- `file_editor`
- `browser_tool_set`
- `glob`
- `grep`
- `planning_file_editor`

## Frontend

**File:** `apps/sms/frontend/src/App.jsx`
**Stack:** React + Vite + Tailwind

### Layout

- **Sticky header**: title "oh-sms", receiving phone number, last-updated timestamp, refresh button
- **Message table**: columns for From, Message, and Time
- **Empty state**: phone icon with "no messages yet"

### Behavior

- Auto-refreshes every 5 seconds
- Manual refresh button in header
- Phone numbers formatted as `(xxx) xxx-xxxx` for US numbers
- Time displayed as relative ("5m ago", "2h ago") for recent messages, absolute for older ones
- Known sender (`ALLOWED_FROM`) highlighted in blue; unknown senders in gray
- Footer shows message count and auto-refresh interval
