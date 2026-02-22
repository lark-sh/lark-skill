---
name: lark
description: Use when working with Lark.sh — a real-time serverless database platform. Covers the Lark CLI, data API, project management, security rules, and best practices.
---

# Lark.sh

Lark is a real-time serverless database. Data is stored as a JSON tree, accessed via REST or WebSocket, with Firebase-style security rules. Each project gets a unique hostname (`{project}.larkdb.net`) for data operations.

**Key concepts:**
- **Project** — top-level container with a unique slug ID, secret key, and settings
- **Database** — named JSON store within a project (e.g. `users`, `chat`)
- **Path** — location in the JSON tree (e.g. `/`, `/users/alice`)
- **Security rules** — JSON5 rules controlling read/write access per path

## CLI (`@lark-sh/cli`)

Install: `npm install -g @lark-sh/cli`

### Getting started

```bash
lark login                        # Authenticate via browser (Google OAuth)
lark config set-project my-app    # Set default project
lark data get mydb /              # Read the whole database
```

### Global options

| Flag | Description |
|------|-------------|
| `--json` | Machine-readable JSON output |
| `--project <id>` | Override the default project |

### Auth

```bash
lark login          # Log in via browser, saves session to ~/.lark/config.json
lark logout         # Clear session
lark whoami         # Show current user (name + email)
```

### Config

```bash
lark config set-project <id>    # Set default project (verified against API)
lark config show                # Show current config
```

Config is stored at `~/.lark/config.json` (mode 0600). Contains: session token, default project, cached admin tokens.

### Projects

```bash
lark projects list                                    # List all projects
lark projects create "My App"                         # Create (name -> slug ID)
lark projects show [id]                               # Show details + secret key
lark projects update [id] --name "New Name"           # Update settings
lark projects update [id] --ephemeral true            # Enable ephemeral mode
lark projects update [id] --auto-create true          # Auto-create databases
lark projects update [id] --firebase-compat true      # Firebase compatibility
lark projects delete [id] --confirm <id>              # Delete (REQUIRES --confirm)
lark projects regenerate-secret [id]                  # New secret key (invalidates old)
```

**Project settings:**
- `ephemeral` — databases are temporary
- `auto_create` — databases created on first access
- `firebase_compat_enabled` — Firebase wire-protocol compatibility
- `firebase_project_id` — Firebase project ID for compat mode

### Databases

```bash
lark databases list                       # List databases (alias: lark db list)
lark databases list --search "chat"       # Filter by ID
lark databases list --limit 10 --offset 5 # Pagination
lark databases create <id>                # Create a database
lark databases delete <id>                # Delete a database
```

### Data operations

Data is a JSON tree. Paths start with `/`. The root `/` is the entire database.

```bash
# Read
lark data get <database> <path>

# Write (overwrite)
lark data set <database> <path> '<json>'
lark data set mydb / '{"key": "value"}' --confirm    # Root requires --confirm!

# Merge (partial update)
lark data update <database> <path> '<json>'

# Append with auto-generated key
lark data push <database> <path> '<json>'

# Delete
lark data delete <database> <path>

# Stdin support (use "-" as value)
cat data.json | lark data set mydb /path -

# Export
lark data export <database> [path]
lark data export mydb / -o backup.json

# Import (overwrites)
lark data import <database> [path] -f <file>
lark data import mydb -f backup.json --confirm        # Root requires --confirm!

# Real-time watch (SSE stream)
lark data watch <database> <path>
```

**Safety:** `set` and `import` to path `/` require `--confirm` because they replace the entire database.

### Security rules

```bash
lark rules get [id]                    # Print current rules
lark rules set [id] -f rules.json     # Set rules from file
echo '{"rules":{".read":true}}' | lark rules set   # Set via stdin
```

Rules use Firebase-style JSON5 syntax controlling `.read` and `.write` at each path.

### Dashboard & billing

```bash
lark dashboard [id]                                  # Metrics overview
lark dashboard --start 2026-02-01 --end 2026-02-15   # Custom date range
lark events [id]                                      # Database events log
lark events --limit 50                                # Paginated
lark billing [id]                                     # Current billing period
lark billing --period 2026-01                         # Specific month
```

Dashboard shows: CCU, peak CCU, bandwidth in/out, reads/writes, permission denials, avg latency, recent events.

## Data API (REST)

Direct HTTP access to database data at `https://{project}.larkdb.net`.

**URL format:** `https://{project}.larkdb.net/{database}{path}.json?v=2&auth={token}`

| Method | Operation |
|--------|-----------|
| `GET` | Read data at path |
| `PUT` | Set (overwrite) data at path |
| `PATCH` | Update (merge) data at path |
| `POST` | Push (append with auto key) |
| `DELETE` | Delete data at path |

**Authentication:** Pass an admin token via `?auth={token}` query parameter. Tokens are JWTs signed with the project's admin secret (HS256), requested via `POST /projects/{id}/admin-token`.

**SSE streaming:** `GET` with `Accept: text/event-stream` header opens a real-time stream of changes.

## Admin API (`db.lark.sh`)

Management API for projects, databases, and metrics. Authenticated via `lark_session` cookie.

**Base URL:** `https://db.lark.sh`

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/me` | GET | Current user info |
| `/projects` | GET | List projects |
| `/projects` | POST | Create project (`{ name }`) |
| `/projects/:id` | GET | Project details |
| `/projects/:id` | PATCH | Update project fields |
| `/projects/:id` | DELETE | Delete project (`{ confirm: id }`) |
| `/projects/:id/regenerate-secret` | POST | New secret key |
| `/projects/:id/admin-token` | POST | Sign admin JWT |
| `/projects/:id/databases` | GET | List databases (`?search=&limit=&offset=`) |
| `/projects/:id/databases` | POST | Create database (`{ id }`) |
| `/projects/:id/databases/:dbId` | DELETE | Delete database |
| `/projects/:id/dashboard` | GET | Metrics + timeseries |
| `/projects/:id/events` | GET | Paginated events |
| `/projects/:id/billing` | GET | Billing info |

**Error format:** Always `{ "error": "message" }` with appropriate HTTP status.

## Common patterns

### Back up a database
```bash
lark data export mydb / -o backup.json
```

### Restore from backup
```bash
lark data import mydb -f backup.json --confirm
```

### Seed initial data
```bash
lark data set mydb /users/alice '{"name": "Alice", "role": "admin"}'
lark data set mydb /users/bob '{"name": "Bob", "role": "user"}'
```

### Watch for changes in a script
```bash
lark data watch mydb /messages --json | while read -r line; do
  echo "$line" | jq '.data'
done
```

### Use in CI/CD with JSON output
```bash
PROJECT_ID=$(lark projects create "CI Test" --json | jq -r '.id')
lark --project "$PROJECT_ID" databases create test-db --json
lark --project "$PROJECT_ID" data set test-db / '{"ready": true}' --confirm --json
```

## IDs and conventions

- **Project IDs** — DNS-compatible slugs derived from the name (e.g. `my-app`)
- **Account IDs** — 16 hex characters
- **Session IDs** — UUID-format hex
- **Secret keys** — `sk_live_` + 64 hex characters
- **Timestamps** — Unix milliseconds everywhere

## Docs & links

- Docs: https://docs.larksh.com (agent-friendly)
- MCP server: https://docs.larksh.com/mcp (provides the SearchLark tool)
- Website: https://lark.sh
- Dashboard: https://dashboard.lark.sh
