# Skills App


![skills screenshot](skills.png)
CRUD editor for OpenHands agent skills. Skills are stored as `SKILL.md` files in the agentskills.io format. Includes a gallery of popular skill templates.

## Ports

| Component | Port |
|-----------|------|
| Frontend  | 4015 |
| Backend   | 4016 |

## Backend

**File:** `apps/skills/backend/main.py`
**Stack:** FastAPI + python-frontmatter
**Dependencies:** `fastapi`, `uvicorn[standard]`, `python-frontmatter`

### Storage

Skills live in `~/.openhands/skills/<name>/SKILL.md`.

Each file has YAML frontmatter + Markdown body:

```markdown
---
name: my-skill
description: What this skill does
license: MIT
compatibility: Requires git
allowed-tools: Bash(git:*) Read Write
metadata:
  key: value
---

Step-by-step instructions in Markdown...
```

### Name Validation

- Regex: `^[a-z0-9]([a-z0-9-]*[a-z0-9])?$`
- Max 64 characters
- No leading/trailing/consecutive hyphens

### Endpoints

#### `GET /api/skills`

Lists all skills. Scans `~/.openhands/skills/` for directories containing `SKILL.md`. Returns a sorted array of skill objects.

#### `GET /api/skills/{name}`

Returns a single skill parsed from its `SKILL.md`.

#### `POST /api/skills` (201)

Body: `{ name, description, license?, compatibility?, allowed_tools?, metadata?, body? }`

Creates a new skill directory and `SKILL.md`. Returns 409 if it already exists. Validates name format and requires non-empty description.

#### `PUT /api/skills/{name}`

Updates an existing skill. If `body.name != name`, renames the directory (returns 409 if new name is taken). Writes the updated frontmatter + body with `sort_keys=False` to preserve field order.

#### `DELETE /api/skills/{name}`

Deletes the entire skill directory via `shutil.rmtree`.

### Frontmatter Handling

Uses `python-frontmatter` library:
- Read: `frontmatter.loads(text)` → `post.get("name")`, `post.content`
- Write: `frontmatter.Post(body, **fm)` → `frontmatter.dumps(post, sort_keys=False)`
- The YAML key `allowed-tools` maps to the API field `allowed_tools`

## Frontend

**File:** `apps/skills/frontend/src/App.jsx`
**Stack:** React + Vite + Tailwind + marked (for Markdown preview)

### Layout

- **Sticky header**: title "oh-skills", path subtitle, "+ New Skill" button
- **Skills grid**: 3-column responsive grid of `SkillCard` components (user's installed skills)
- **Popular Skills section**: curated templates grouped by category, with "Add" buttons

### Components

#### `SkillCard`

Displays skill name, description (3-line clamp), license/compatibility/allowed-tools tags. Edit (pencil) and delete (trash) buttons.

#### `SkillEditorModal`

Full-screen modal for creating or editing a skill:

- **Name** (disabled when editing existing skill)
- **Description** (textarea, character count)
- **License**, **Compatibility**, **Allowed Tools** inputs
- **Metadata** — dynamic key-value pair editor (`MetadataEditor` component): add/remove rows
- **Body** — tabbed Markdown editor (`MarkdownEditor` component): "Write" tab with textarea, "Preview" tab rendering HTML via `marked.parse()`

#### `MetadataEditor`

Renders a list of key/value input rows with add (+) and remove (×) buttons.

### Popular Skills

`src/popularSkills.js` exports:
- `POPULAR_SKILLS` — array of template objects with `{ name, label, description, body, category, license? }`
- `CATEGORIES` — ordered list of category names: "Official", "Coding Workflow", "Language & Framework", "DevOps"

Each category has a styled pill badge. Skills already installed show "Added ✓" instead of the add button. Clicking "Add" pre-fills the editor modal with the template data.

### Behavior

- Skill list is sorted alphabetically
- Delete requires `confirm()` dialog
- Creating a skill: fills in form, click "Create Skill" → `POST /api/skills` → adds to list
- Editing: opens modal pre-filled with current data → `PUT /api/skills/{name}` → updates list
- From template: pre-fills name, description, license, and body from the template
