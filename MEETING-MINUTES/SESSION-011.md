# Session 011

Please read [SKILL.md](../.claude/skills/explore-specs/SKILL.md) to understand the process used to create this file.

**Session Focus:** Backend development enablement - clarifying implementation details needed to start backend development for the v1 feature set.

---

## Question #1: Search API Design (Outstanding from SESSION-010)

SESSION-010 Q#7 established basic text search for workspace list (by title) and chat list (by title). The Post-Session Note #2 flagged that API design is required.

How should the search query be passed to the API endpoints?

### Answer

Add an optional `q` query parameter to existing list endpoints:
- `GET /api/workspaces?q=blog` returns workspaces with "blog" in the title
- `GET /api/workspaces/:id/chats?q=typescript` returns matching chats

Case-insensitive substring match. Empty or missing `q` returns all results. This follows common REST patterns used by GitHub API, Linear API, and similar tools.

---

## Question #2: SQLite Configuration for Bun

Given this is a single-user application with concurrent operations (runner workers + API + background jobs all in one process), what SQLite configuration should be used?

### Answer

Use Bun's built-in `bun:sqlite` with a single `Database` instance shared across the application. On startup, configure with:

- `PRAGMA journal_mode = WAL;` (better concurrent read performance)
- `PRAGMA synchronous = NORMAL;` (safe for WAL, better performance than FULL)
- `PRAGMA busy_timeout = 5000;` (5 second wait on lock contention)

Single-user + single-process means connection pooling is unnecessary. Bun's SQLite handles concurrent reads naturally with WAL. Writes are serialized by SQLite automatically. Use transactions for multi-step operations (e.g., create task + queue item + activity log).

---

## Question #3: Subprocess Management for CLI Execution

The runner needs to spawn CLI processes with environment variables, track running processes for cancellation, and kill subprocesses on task/workspace deletion. How should subprocess tracking work?

### Answer

Maintain in-memory Maps for subprocess tracking:
- `Map<string, Subprocess>` keyed by task ID for task processing
- `Map<string, Subprocess>` keyed by chat ID for chat processing

When cancellation is requested:
1. Look up the subprocess in the map
2. Call `subprocess.kill()` (Bun's subprocess API)
3. Remove from map after process exits

For workspace deletion: iterate all maps, find entries matching the workspace, kill those subprocesses first, then proceed with cascade delete.

---

## Question #4: Runner Worker Implementation Pattern

How should the worker pattern be implemented for the runner that processes multiple workspaces concurrently?

### Answer

Single polling loop with dynamic worker spawning:

1. One main runner loop polls every `MALAMAR_RUNNER_POLL_INTERVAL` (default 1000ms)
2. Query for all workspaces with `queued` queue items (respecting the pickup algorithm)
3. For each workspace with work AND no active worker, spawn an async worker function
4. Track active workers in a `Set<string>` (workspace IDs currently processing)
5. Worker function processes the task loop, removes itself from the Set when done

No artificial limit on concurrent workers. Workers are async functions, not separate threads/processes - leveraging Bun's event loop.

---

## Question #5: CLI Output Detection and Error Classification

How should the runner detect when the CLI finishes and classify the result?

### Answer

After subprocess exits, check in this order:

1. **Exit code non-zero**: System comment "CLI exited with code {code}. {stderr if available}"
2. **Output file missing**: System comment "CLI completed but output file was not created at {path}"
3. **Output file empty**: System comment "CLI completed but output file was empty"
4. **JSON parse failure**: System comment "CLI output was not valid JSON: {parse error}"
5. **Schema validation failure**: System comment "CLI output structure was invalid: {validation error}"
6. **Success**: Parse actions and execute

All error cases trigger the standard error flow (stop loop, system comment emits queue item for retry).

---

## Question #6: SSE Implementation Pattern

How should SSE connections and event broadcasting be implemented?

### Answer

EventEmitter + connection registry pattern:

- Create a singleton `EventEmitter` instance for pub/sub
- Maintain a `Set<Response>` of active SSE connections
- On new SSE request: add response to Set, send initial `:ok` comment, set up disconnect cleanup
- When emitting events: `eventEmitter.emit(eventType, payload)` triggers a handler that iterates the Set and writes to each connection
- On client disconnect: remove from Set
- Event format per SSE spec: `event: {type}\ndata: {json}\n\n`
- Include `retry: 3000` on connection start for auto-reconnect hint

---

## Question #7: API Error Response Format

What format should API error responses use?

### Answer

Simple JSON with code and message:

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Task not found"
  }
}
```

HTTP status codes for categories:
- `400` - Validation errors (code: `VALIDATION_ERROR`, message includes field details)
- `404` - Resource not found (code: `NOT_FOUND`)
- `409` - Conflict (code: `CONFLICT`, e.g., duplicate agent name)
- `500` - Server error (code: `INTERNAL_ERROR`)

Frontend can display `error.message` directly.

---

## Question #8: Background Job Scheduling Mechanism

How should scheduled background jobs (Cleanup daily, CLI Health Check every 5 minutes) be implemented?

### Answer

Simple setInterval with startup trigger:

- **CLI Health Check**: `setInterval(checkCliHealth, 5 * 60 * 1000)` + run once on startup
- **Cleanup Job**: `setInterval(runCleanup, 24 * 60 * 60 * 1000)` + run once on startup
- Store last run timestamp in memory (not persisted - if Malamar restarts, jobs run again which is fine)
- Jobs are async functions that don't block the event loop

No need for cron parsing or complex scheduling. The "run on startup" ensures cleanup happens even if the interval drifts.

---

## Question #9: Graceful Shutdown Handling

How should graceful shutdown be handled when Malamar is stopped (SIGTERM/SIGINT)?

### Answer

Signal handler with subprocess cleanup:

1. Register handlers for `SIGTERM` and `SIGINT`
2. On signal:
   - Stop accepting new queue pickups (set a shutdown flag)
   - Kill all active CLI subprocesses (iterate the subprocess maps)
   - Wait briefly (e.g., 1 second) for subprocesses to exit
   - Close SSE connections (clients will auto-reconnect, see server gone)
   - Close database connection (SQLite WAL will checkpoint automatically)
   - Exit process
3. No need to update queue items - startup recovery handles `in_progress` items by resetting them to `queued`

---

## Question #10: File Serving for Chat Attachments

Should the backend serve attachment files via API?

### Answer

No. Attachments are only for CLI agents to read, not displayed in the UI. No API endpoint for serving attachment files.

---

## Question #11: API Response Pagination

Should list endpoints support pagination?

### Answer

No pagination for v1. Return all items in list endpoints. Reasons:

- Single-user app with limited data volume
- Comments/logs per task are typically manageable (tens, not thousands)
- Simplifies frontend implementation
- Can add pagination later if real-world usage shows need

Apply same principle to all list endpoints: workspaces, agents, tasks, comments, logs, chats, messages.

---

## Question #12: Nanoid Generation Configuration

What nanoid configuration should be used for ID generation?

### Answer

Default nanoid (21 characters, URL-safe):

- Use `nanoid()` with default settings: 21 characters, alphabet `A-Za-z0-9_-`
- Matches the mock user ID length (21 zeros)
- URL-safe for use in API paths like `/api/tasks/:id`
- Sufficient collision resistance for single-user app (and future multi-user)
- Use the `nanoid` npm package

---

## Question #13: Queue Item Creation Trigger Points

What actions should create or update queue items?

### Answer

**Create new queue item (status: `queued`):**
- Task created (via API) → create queue item
- Task moved from Done/In Review to Todo (via API) → create queue item

**Create or update queue item on comment:**
- Any comment added (user/agent/system) to a task → create queue item or update `updated_at` on existing `queued` item
- Exception: If task is in "Done" status, no queue item is created

**Update existing queue item (`updated_at` bump):**
- If a `queued` item already exists for that task, don't create duplicate - just update `updated_at`

**Create queue item from system comment (error retry):**
- System comment added (error case) → create/update queue item

---

## Question #14: Activity Log Event Types

What is the complete list of activity log event types?

### Answer

| Event Type | Actor Type | Metadata |
|------------|------------|----------|
| `task_created` | user | - |
| `status_changed` | user/agent/system | `{ old_status, new_status }` |
| `comment_added` | user/agent/system | - |
| `agent_started` | agent | `{ agent_name }` |
| `agent_finished` | agent | `{ agent_name, action_type }` where action_type is "skip", "comment", or "in_review" |
| `task_cancelled` | user | - |
| `task_prioritized` | user | - |
| `task_deprioritized` | user | - |

Note: `comment_added` doesn't include comment content (that's in the comments table). `agent_finished` includes action summary so the log is meaningful standalone.

---

## Question #15: Chat Queue Processing Model

How should chat queue processing be implemented?

### Answer

Separate chat processor with immediate pickup:

- Maintain a separate polling loop for chat queue (same interval as task runner)
- When user sends a message: create `chat_queue` item with status `queued`, return API response immediately
- Chat processor picks up `queued` chat items
- No "one per workspace" limit - multiple chats can process concurrently
- Track active chat subprocesses in separate `Map<string, Subprocess>` keyed by chat_id (for cancellation)
- On completion: parse output, execute actions, add agent message to chat, emit SSE events
- On error: add system message to chat, mark queue item failed

Simpler than task processing (no multi-agent loop), but uses same subprocess and error handling patterns.

---

## Question #16: Malamar Agent Action Execution

When an agent returns multiple actions, how should they be executed?

### Answer

Execute all, collect errors:

- Execute actions in array order
- Each action executes independently (failure of one doesn't stop others)
- Collect all errors and add a single system message summarizing failures
- Example: If `create_agent` fails due to duplicate name but `rename_chat` succeeds, system message says "Action failed: create_agent - Agent name 'Reviewer' already exists"
- The agent's conversational `message` is always added to chat (regardless of action success/failure)

---

## Question #17: CLI Health Check Implementation

What exactly constitutes a successful health check?

### Answer

Exit code zero + any output:

- Run CLI with minimal prompt (e.g., `claude --prompt "Reply OK"` or equivalent per CLI)
- Health check passes if:
  1. Process exits with code 0
  2. stdout is non-empty (any response, doesn't need to match "OK")
- Health check fails if:
  - Binary not found → error: "Binary not found in PATH"
  - Non-zero exit code → error: "CLI exited with code {code}"
  - Empty output → error: "CLI returned empty response"
  - Timeout (30 seconds) → error: "CLI health check timed out"
- Store status and error message in memory
- Expose via `GET /api/health/cli` endpoint

---

## Question #18: First Startup Detection

How should the backend detect this is a first startup to trigger sample workspace creation?

### Answer

Check for data directory existence:

- On startup, check if `~/.malamar` directory exists (or `MALAMAR_DATA_DIR` if configured)
- If directory does NOT exist:
  1. Set in-memory flag `isFirstLaunch = true`
  2. Create the directory
  3. Initialize database, run migrations
  4. Create sample workspace with agents
- If directory exists:
  1. `isFirstLaunch = false`
  2. Open existing database, run migrations
  3. No sample workspace creation

This ensures sample workspace is only created once, ever. Even if user deletes all workspaces, directory still exists, so no recreation.

---

## Question #19: Environment Variable Injection for CLI

How should environment variables be merged when spawning CLI subprocesses?

### Answer

Inherit system env + override with user settings:

- Start with `process.env` (inherit all system environment variables)
- Merge in user-configured env vars from CLI settings (user values override system values if same key)
- Pass merged env to `Bun.spawn()` via the `env` option
- Example: If system has `PATH=/usr/bin` and user configures `ANTHROPIC_API_KEY=xxx`, CLI gets both
- If user configures `PATH=/custom/path`, it overrides system PATH

This ensures CLIs work correctly (they often need PATH, HOME, etc.) while allowing user customization.

---

## Question #20: Database Migration Strategy

How should database migrations be implemented?

### Answer

Sequential numbered SQL files bundled into the Bun binary:

- Store migrations as embedded files in the binary
- Naming: `001_initial_schema.sql`, `002_add_chat_tables.sql`, etc.
- Track applied migrations in a `_migrations` table: `(version INTEGER PRIMARY KEY, applied_at DATETIME)`
- On startup:
  1. Create `_migrations` table if not exists
  2. Get max applied version
  3. Run all migrations with version > max, in order
  4. If any migration fails, panic with error message
- Each migration file contains raw SQL statements separated by `;`
- No rollback support (manual restore from backup)

---

## Question #21: `last_activity_at` Update Mechanism

How should workspace `last_activity_at` be updated?

### Answer

Inline update in transaction:

- When any activity occurs (task created, comment added, status changed, agent execution, etc.), include a workspace `last_activity_at` update in the same database transaction
- Example: Creating a task transaction includes:
  1. `INSERT INTO tasks ...`
  2. `INSERT INTO task_queue ...`
  3. `INSERT INTO task_logs ...`
  4. `UPDATE workspaces SET last_activity_at = NOW() WHERE id = ?`

Simple, consistent, no eventual consistency issues. Small overhead per operation (one extra UPDATE).

---

## Question #22: HTTP Server Framework

What HTTP server approach should be used with Bun?

### Answer

**Hono** - lightweight, fast web framework designed for edge/Bun environments:

- Simple routing: `app.get('/api/workspaces', handler)`
- Middleware support for error handling, logging
- TypeScript-first with good type inference
- Works with Bun's native APIs (including SSE via native Response)
- Popular choice for Bun projects, well-documented

Provides structure without heavy abstraction.

---

## Question #23: Request Validation Approach

How should the backend validate incoming API requests?

### Answer

Zod schemas with Hono integration:

- Define Zod schemas for each request body type
- Use `@hono/zod-validator` middleware for automatic validation
- On validation failure, return 400 with error format:
  ```json
  {
    "error": {
      "code": "VALIDATION_ERROR",
      "message": "title is required; description must be a string"
    }
  }
  ```
- Schemas double as TypeScript types (single source of truth)
- Example: `CreateWorkspaceSchema = z.object({ title: z.string().min(1), description: z.string().optional() })`

---

## Question #24: Static File Serving for Frontend

How should the backend serve the frontend static files?

### Answer

Hono static middleware with embedded files:

- At build time: embed frontend `dist/` files into the binary (Bun supports this)
- At runtime: serve embedded files for non-API routes
- Route priority:
  1. `/api/*` → API handlers
  2. Known static extensions (`.js`, `.css`, `.png`, etc.) → serve from embedded assets
  3. All other routes → serve `index.html` (SPA fallback for client-side routing)
- Set appropriate `Content-Type` headers based on file extension
- Set cache headers for assets (e.g., `Cache-Control: public, max-age=31536000` for hashed filenames)

---

## Question #25: Test Email Endpoint Implementation

What should the test email contain and how should the endpoint behave?

### Answer

Simple verification email with sync response:

- Request body: empty (uses saved Mailgun settings)
- Validate settings exist (API key, domain, from/to emails) → 400 if missing
- Send email with:
  - Subject: "Malamar Test Email"
  - Body: "This is a test email from Malamar. If you received this, your email notifications are configured correctly."
- Wait for Mailgun API response (don't fire-and-forget for test)
- On success: return 200 `{ "success": true }`
- On Mailgun error: return 400 `{ "error": { "code": "EMAIL_FAILED", "message": "Mailgun error: {details}" } }`

---

## Question #26: CLI Adapter Command Templates

What are the exact command templates for all four CLIs?

### Answer

**Claude Code** (supports JSON schema):
```shell
claude --dangerously-skip-permissions \
  --output-format json \
  --json-schema '{...schema...}' \
  --prompt "Read the file at {input_path} and follow the instruction autonomously."
```

**Gemini CLI** (schema in prompt):
```shell
gemini --prompt "Read the file at {input_path} and follow the instruction autonomously."
```

**Codex CLI** (schema in prompt):
```shell
codex --prompt "Read the file at {input_path} and follow the instruction autonomously."
```

**OpenCode** (schema in prompt):
```shell
opencode --prompt "Read the file at {input_path} and follow the instruction autonomously."
```

For CLIs without `--json-schema`, embed the JSON format instruction at the end of the input file's "Output Instruction" section. All CLIs use `cwd` option set to the working directory.

*Note: Actual flags may need verification against current CLI versions at implementation time.*

---

## Question #27: Logging Implementation

How should logging be implemented?

### Answer

Simple custom logger:

- Create a logger module that respects `MALAMAR_LOG_LEVEL` and `MALAMAR_LOG_FORMAT`
- Log levels: `debug`, `info`, `warn`, `error` (each includes higher levels)
- Text format: `[2025-01-18T10:30:00Z] [INFO] Message here { context }`
- JSON format: `{"timestamp":"2025-01-18T10:30:00Z","level":"info","message":"Message here","context":{}}`
- API: `logger.info("message", { optional: "context" })`, `logger.error("message", { error })`
- Write to stdout (text/json based on config)
- No external logging library needed for this scope

---

## Question #28: Configuration Loading Priority

What is the exact loading order for configuration?

### Answer

Env vars override CLI flags override defaults:

- Loading order (later wins):
  1. Hardcoded defaults (e.g., port 3456)
  2. CLI flags (`--port 8080`)
  3. Environment variables (`MALAMAR_PORT=9000`)
- Parse CLI flags first using Bun's `process.argv` or a minimal parser
- Then check for corresponding env vars and override if present
- Example: `--port 8080` + `MALAMAR_PORT=9000` → uses 9000
- Expose final config via `malamar config` command

---

## Question #29: `malamar doctor` Command Implementation

What should `malamar doctor` check and output?

### Answer

Checklist with pass/fail status:

```
Malamar Doctor

✓ Data directory: ~/.malamar (exists, writable)
✓ Database: malamar.db (accessible, migrations current)
✓ Configuration: valid

CLI Status:
✓ Claude Code: healthy (claude found at /usr/local/bin/claude)
✗ Gemini CLI: unhealthy (binary not found in PATH)
✓ Codex CLI: healthy (codex found at /usr/local/bin/codex)
✗ OpenCode: unhealthy (test prompt failed)

2 of 4 CLIs available
```

Exit code: 0 if all critical checks pass (data dir, database), non-zero if critical failure. CLI availability is informational, not critical (user might only need one CLI).

---

## Question #30: Working Directory Management for Tasks

When/how is the working directory created for Temp Folder mode?

### Answer

Create on first agent execution, no cleanup:

**Temp Folder mode:**
- Directory path: `/tmp/malamar_task_{task_id}` (using configured temp dir)
- Create directory when the first agent for that task is about to execute
- If directory already exists (from previous loop), reuse it
- No automatic cleanup (let OS handle `/tmp`)

**Static Directory mode:**
- Use the path from workspace settings as-is
- Don't create it if it doesn't exist (user's responsibility)
- Log warning if directory doesn't exist at execution time

Pass the directory as `cwd` to `Bun.spawn()`.

---

## Question #31: CORS Handling

How should CORS be handled?

### Answer

No CORS handling needed. The frontend uses Vite's proxy feature during development to forward `/api/*` requests to `localhost:3456`. The frontend always calls `/api/...` relative URLs, making behavior consistent between development and production.

---

## Question #32: Enforcing `rename_chat` First Response Only

How should the "first response only" restriction for `rename_chat` be enforced?

### Answer

Check message count before executing action:

- When processing chat output actions, before executing `rename_chat`:
  1. Query `SELECT COUNT(*) FROM chat_messages WHERE chat_id = ? AND role = 'agent'`
  2. If count > 0 (meaning there's already an agent response), ignore the `rename_chat` action silently
  3. If count = 0, execute the rename
- Silent ignore (not an error) because:
  - Agent instructions tell agents to rename on first response
  - If agent mistakenly tries later, no harm in ignoring
  - No need for system error message
