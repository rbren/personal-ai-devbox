# Secrets App


![secrets screenshot](secrets.png)
Key-value secret manager. Stores secrets in `~/.openhands/remote/secrets.json` as a flat `{ key: value }` JSON object. These secrets are injected into new OpenHands conversations by the SMS and Scheduled apps.

## Ports

| Component | Port |
|-----------|------|
| Frontend  | 4009 |
| Backend   | 4010 |

## Backend

**File:** `apps/secrets/backend/main.py`
**Stack:** FastAPI
**Dependencies:** `fastapi`, `uvicorn[standard]`

### Storage

File: `~/.openhands/remote/secrets.json`
Format: `{ "KEY_NAME": "value", ... }`

Parent directory is created automatically if missing.

### Endpoints

#### `GET /api/secrets`

Returns `{ secrets: [{ key, value }, ...] }`.

#### `PUT /api/secrets`

Body: `{ secrets: [{ key, value }, ...] }`. Replaces the entire secrets file. Deduplicates by key (first occurrence wins). Returns 400 if any key is empty.

#### `DELETE /api/secrets/{key}`

Deletes a single secret by key. Returns 404 if not found.

## Frontend

**File:** `apps/secrets/frontend/src/App.jsx`
**Stack:** React + Vite + Tailwind

### Layout

- **Sticky header**: title "oh-secrets", file path subtitle, "saving…" indicator
- **Secrets table**: columns for Key, Value, and a delete button (visible on hover)
- **Add form** below the table: KEY input, value input (type=password), "+ add" button

### Behavior

- **Inline editing**: click a key or value to edit in place. Changes are committed on blur or Enter, cancelled on Escape.
- **Reveal/hide**: each value row has an eye toggle. Values are hidden by default (shown as `●●●●`). Clicking the eye reveals the actual value.
- **Delete**: trash icon appears on row hover. Deletes are immediate (no confirmation).
- **Add**: type a new key and value in the bottom form, press Enter or click "+ add". Validates that key is non-empty and not a duplicate.
- All mutations send the full secrets array via `PUT /api/secrets` and replace local state with the response — the backend is the source of truth.

### API

Calls are made directly in `App.jsx` via two functions:
- `apiGet()` → `GET /apps/secrets/api/secrets`
- `apiPut(secrets)` → `PUT /apps/secrets/api/secrets`
