# Malamar Technical Design

This document covers the technical implementation details for Malamar. For product specifications (what and why), see [SPECS.md](./SPECS.md).

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Data Model](#data-model)
- [Runner & Queue Mechanics](#runner--queue-mechanics)
- [CLI Adapters](#cli-adapters)
- [API & Real-Time](#api--real-time)
- [User Interface Implementation](#user-interface-implementation)
- [Background Jobs](#background-jobs)
- [Configuration Reference](#configuration-reference)
- [Operations](#operations)
- [Testing](#testing)
- [Development Tooling](#development-tooling)

---

## Architecture Overview

### Tech Stack

**Backend:**
- TypeScript with Bun runtime
- Hono web framework with Zod validation
- RESTful API
- Background jobs (runner, cleanup, health check)
- SSE endpoint for real-time updates
- SQLite database with WAL mode

**Frontend:**
- TypeScript with React (via Vite)
- React Query for server-state management
- Zustand for client-only state (when needed)
- React Hook Form with Zod resolver for forms
- shadcn/ui component library with Tailwind CSS
- Lucide React for icons
- @dnd-kit for drag-and-drop (agent reordering)
- react-markdown with remark-gfm and rehype-highlight for markdown rendering
- next-themes for theme switching
- Bun as the build tool/runtime
- Mobile-first responsive design

**Distribution:**
- Single executable binary, or run via `bunx`/`npx`
- Can be installed with Homebrew

### High-Level System Design

Malamar runs as a single process with:
- An embedded SQLite database for persistence
- A web server serving both the REST API and static frontend files
- Background jobs running in-process (runner, cleanup, health check)
- An in-process pub/sub event emitter for SSE broadcasting

No external dependencies (Redis, external databases, message queues) are required.

---

## Data Model

### Database

Malamar uses SQLite as its database, stored at `~/.malamar/malamar.db` (configurable via `MALAMAR_DATA_DIR`).

**SQLite Configuration:**

Use Bun's built-in `bun:sqlite` with a single `Database` instance shared across the application. On startup, configure with:

- `PRAGMA journal_mode = WAL;` (better concurrent read performance)
- `PRAGMA synchronous = NORMAL;` (safe for WAL, better performance than FULL)
- `PRAGMA busy_timeout = 5000;` (5 second wait on lock contention)

Single-user + single-process means connection pooling is unnecessary. Bun's SQLite handles concurrent reads naturally with WAL. Writes are serialized by SQLite automatically. Use transactions for multi-step operations (e.g., create task + queue item + activity log).

### Database Migrations

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

### ID Generation

Use `nanoid()` with default settings:
- 21 characters, alphabet `A-Za-z0-9_-`
- URL-safe for use in API paths like `/api/tasks/:id`
- Sufficient collision resistance for single-user app (and future multi-user)
- Use the `nanoid` npm package

The mock user ID is `000000000000000000000` (21 zeros, a valid nanoid).

### Entity Schemas

#### Workspaces

```
workspaces
├── id: string (nanoid, primary key)
├── title: string
├── description: string (workspace-level instruction for agents)
├── working_directory_mode: enum ("static" | "temp")
├── working_directory_path: string? (only when mode is "static")
├── auto_delete_done_tasks: boolean (default: true)
├── retention_days: integer (default: 7, 0 to disable)
├── notify_on_error: boolean (default: true)
├── notify_on_in_review: boolean (default: true)
├── last_activity_at: datetime
├── created_at: datetime
└── updated_at: datetime
```

#### Agents

```
agents
├── id: string (nanoid, primary key)
├── workspace_id: string (foreign key)
├── name: string
├── instruction: string (system instruction for the agent)
├── cli_type: enum ("claude" | "gemini" | "codex" | "opencode")
├── order: integer (unique within workspace)
├── created_at: datetime
└── updated_at: datetime
```

#### Tasks

```
tasks
├── id: string (nanoid, primary key)
├── workspace_id: string (foreign key)
├── summary: string (title)
├── description: string (markdown content)
├── status: enum ("todo" | "in_progress" | "in_review" | "done")
├── created_at: datetime
└── updated_at: datetime
```

#### Task Comments

```
task_comments
├── id: string (nanoid, primary key)
├── task_id: string (foreign key)
├── workspace_id: string (foreign key)
├── user_id: string? (null if system or agent comment)
├── agent_id: string? (null if system or user comment)
├── content: string (markdown)
├── created_at: datetime
└── updated_at: datetime
```

#### Task Activity Logs

```
task_logs
├── id: string (nanoid, primary key)
├── task_id: string (foreign key)
├── workspace_id: string (foreign key)
├── event_type: string
├── actor_type: enum ("user" | "agent" | "system")
├── actor_id: string? (nullable for system events)
├── metadata: json
├── created_at: datetime
```

**Activity Log Event Types:**

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

#### Task Event Queue

```
task_queue
├── id: string (nanoid, primary key)
├── task_id: string (foreign key)
├── workspace_id: string (foreign key)
├── status: enum ("queued" | "in_progress" | "completed" | "failed")
├── is_priority: boolean (default: false)
├── created_at: datetime
└── updated_at: datetime
```

#### Chats

```
chats
├── id: string (nanoid, primary key)
├── workspace_id: string (foreign key)
├── agent_id: string? (foreign key, NULL if chatting with Malamar agent)
├── cli_type: string? (NULL = respect agent's CLI or Malamar default)
├── title: string (default: "Untitled chat")
├── created_at: datetime
└── updated_at: datetime
```

#### Chat Messages

```
chat_messages
├── id: string (nanoid, primary key)
├── chat_id: string (foreign key)
├── role: enum ("user" | "agent" | "system")
├── message: string (the conversational text)
├── actions: json? (null for user/system, optional array for agent)
├── created_at: datetime
```

#### Chat Queue

```
chat_queue
├── id: string (nanoid, primary key)
├── chat_id: string (foreign key)
├── workspace_id: string (foreign key)
├── status: enum ("queued" | "in_progress" | "completed" | "failed")
├── created_at: datetime
└── updated_at: datetime
```

#### Global Settings

```
settings
├── key: string (primary key)
└── value: json
```

Settings keys include:
- `mailgun_api_key`
- `mailgun_domain`
- `mailgun_from_email`
- `mailgun_to_email`
- `notify_on_error` (global default)
- `notify_on_in_review` (global default)
- `cli_settings` (JSON object with per-CLI configuration)

### Cascade Deletes

When entities are deleted, associated data is cascade deleted:

- **Workspace deleted**: All agents, tasks, comments, activity logs, task queue items, chats, chat messages, and chat queue items
- **Task deleted**: All comments, activity logs, and queue items
- **Chat deleted**: All chat messages and chat queue items (attachment directory left for OS cleanup)
- **Agent deleted**: Comments and chats retain `agent_id` for reference; comments display "(Deleted Agent)", chats are switched to Malamar agent

---

## Runner & Queue Mechanics

### Runner Overview

The runner is the main background job that processes tasks. It runs as a continuous loop with configurable polling interval (default: 1000ms).

### Concurrent Workers

Single polling loop with dynamic worker spawning:

1. One main runner loop polls every `MALAMAR_RUNNER_POLL_INTERVAL` (default 1000ms)
2. Query for all workspaces with `queued` queue items (respecting the pickup algorithm)
3. For each workspace with work AND no active worker, spawn an async worker function
4. Track active workers in a `Set<string>` (workspace IDs currently processing)
5. Worker function processes the task loop, removes itself from the Set when done

No artificial limit on concurrent workers. Workers are async functions, not separate threads/processes - leveraging Bun's event loop.

### Queue Item Creation Triggers

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

### Queue Pickup Algorithm

Per workspace, the runner picks up queue items in this order:

1. **Status filter**: Only consider items where task status is "Todo" or "In Progress"
2. **Priority flag**: Items with `is_priority: true` and status `queued`
3. **Most recently processed**: Find the task with the most recent `completed` or `failed` queue item, pick its `queued` item
4. **LIFO fallback**: Pick the most recently updated `queued` item

### Agent Execution (Just-in-Time Snapshot)

Before executing each agent:
1. Query for the next agent by finding the smallest `order` value greater than current
2. Fetch agent data (instruction, name, CLI) right before execution
3. Changes to agents take effect immediately for subsequent agents

**Mid-Flight Changes:** If workspace settings (like working directory) are changed while a task is actively being processed, the currently running agent continues with the old settings. Subsequent agents in the loop pick up the new settings. This is an accepted risk that applies to both Malamar agent actions and manual user changes.

### Loop Flow

```
1. Pick queue item
2. Move task to "In Progress" (if in "Todo")
3. Auto-demote other "In Progress" tasks to "Todo"
4. For each agent (by order):
   a. Fetch agent data
   b. Create input file
   c. Invoke CLI
   d. Read output file
   e. Execute actions (skip/comment/change_status)
   f. If change_status to "In Review", stop loop
5. After all agents:
   - If any comment added: retrigger loop from first agent
   - If all skipped: move task to "In Review"
```

**Workspace With No Agents:** When a workspace has zero agents, tasks immediately move to "In Review" (no agents = all skip). Chats can still happen with the Malamar agent to recreate agents.

### Error Handling

After subprocess exits, check in this order:

1. **Exit code non-zero**: System comment "CLI exited with code {code}. {stderr if available}"
2. **Output file missing**: System comment "CLI completed but output file was not created at {path}"
3. **Output file empty**: System comment "CLI completed but output file was empty"
4. **JSON parse failure**: System comment "CLI output was not valid JSON: {parse error}"
5. **Schema validation failure**: System comment "CLI output structure was invalid: {validation error}"
6. **Success**: Parse actions and execute

All error cases trigger the standard error flow:
- Write System comment with error details
- Stop current loop
- System comment emits task event → new queue item
- Task stays in current status (no move to "In Review")
- Natural retry via queue (no explicit backoff)

**CLI Becomes Unhealthy Mid-Task:**
- No pre-check of CLI health before invocation - just try to run it
- If CLI fails, normal error handling kicks in (system comment, retry via queue)
- If CLI stays unhealthy, task keeps retrying with accumulating error comments
- User sees errors and can fix the CLI or reassign agent to a different CLI

### Subprocess Management

Maintain in-memory Maps for subprocess tracking:
- `Map<string, Subprocess>` keyed by task ID for task processing
- `Map<string, Subprocess>` keyed by chat ID for chat processing

When cancellation is requested:
1. Look up the subprocess in the map
2. Call `subprocess.kill()` (Bun's subprocess API)
3. Remove from map after process exits

For workspace deletion: iterate all maps, find entries matching the workspace, kill those subprocesses first, then proceed with cascade delete.

### Task Cancellation

When a user cancels an "In Progress" task:
1. CLI subprocess is killed immediately
2. Task moves to "In Review" (prevents immediate re-pickup)
3. System comment added: "Task cancelled by user"
4. User investigates and fixes description/instructions
5. When user comments, task moves back to Todo → In Progress
6. User can prioritize if immediate processing needed

### Deletion While Processing

**Workspace Deletion:**
- If any task in the workspace is being processed, kill the CLI subprocess first
- Then proceed with cascade delete
- Same logic applies to chat processing - kill any active chat CLI subprocess before deleting

**Task Deletion:**
- Delete action IS available for In Progress tasks
- Kill subprocess first, then cascade delete
- One-click removal for stuck or problematic tasks

### Startup Recovery

On startup:
- Find any queue items (task or chat) with status "in_progress"
- Reset them to "queued"
- Runner picks them up naturally
- Tasks stay "In Progress" status, just re-queued for processing
- Clean crash recovery without special logic

### Graceful Shutdown

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

### Workspace Activity Tracking

When any activity occurs (task created, comment added, status changed, agent execution, etc.), include a workspace `last_activity_at` update in the same database transaction:

```sql
-- Example: Creating a task transaction includes:
INSERT INTO tasks ...
INSERT INTO task_queue ...
INSERT INTO task_logs ...
UPDATE workspaces SET last_activity_at = NOW() WHERE id = ?
```

Simple, consistent, no eventual consistency issues. Small overhead per operation (one extra UPDATE).

---

## CLI Adapters

### Adapter Implementation

Each supported CLI has an adapter with:
- Command template
- Supported flags
- Response format enforcement method

**Locked Down Templates:** CLI command templates are intentionally locked down for stability. Users can customize binary paths and environment variables via CLI Settings, but not the command templates themselves. CLI adapters are hardcoded.

### CLI Invocation

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

**Environment Variable Injection:**

Inherit system env + override with user settings:

- Start with `process.env` (inherit all system environment variables)
- Merge in user-configured env vars from CLI settings (user values override system values if same key)
- Pass merged env to `Bun.spawn()` via the `env` option
- Example: If system has `PATH=/usr/bin` and user configures `ANTHROPIC_API_KEY=xxx`, CLI gets both
- If user configures `PATH=/custom/path`, it overrides system PATH

This ensures CLIs work correctly (they often need PATH, HOME, etc.) while allowing user customization.

### Response Format Enforcement

- **CLIs with schema support** (Claude Code): Use native `--json-schema` flag
- **CLIs without schema support**: Embed JSON format instruction in the prompt

### Task Context Passing

#### Task Input File

Location: `/tmp/malamar_task_{task_id}.md` (overwritten for each agent execution)

```markdown
# Malamar Context
You are being orchestrated by Malamar, a multi-agent workflow system.
{workspace instruction here}

# Your Role
{agent instruction here}

## Other Agents in This Workflow
- Planner
- Implementer
- Reviewer
- Approver

# Task
## Summary
{task summary}

## Description
{task description}

## Comments

​```json
{"author": "Planner", "agent_id": "abc123xyz", "content": "## My Plan\n\n1. First step\n2. Second step", "created_at": "2025-01-17T10:00:00Z"}
{"author": "User", "user_id": "000000000000000000000", "content": "Looks good, proceed!", "created_at": "2025-01-17T10:05:00Z"}
{"author": "System", "content": "Error: CLI timeout after 30s", "created_at": "2025-01-17T10:10:00Z"}
​```

## Activity Log

​```json
{"event_type": "created", "actor_type": "user", "actor_id": "000000000000000000000", "created_at": "2025-01-17T09:55:00Z"}
{"event_type": "status_changed", "actor_type": "system", "metadata": {"old_status": "todo", "new_status": "in_progress"}, "created_at": "2025-01-17T09:56:00Z"}
{"event_type": "agent_started", "actor_type": "agent", "actor_id": "abc123xyz", "metadata": {"agent_name": "Planner"}, "created_at": "2025-01-17T09:56:01Z"}
{"event_type": "comment_added", "actor_type": "agent", "actor_id": "abc123xyz", "created_at": "2025-01-17T10:00:00Z"}
​```

# Output Instruction
Write your response as JSON to: /tmp/malamar_output_{random_nanoid}.json
```

**Format Notes:**
- Comments and activity logs in JSONL format (one JSON per line) inside code blocks
- Prevents markdown in comments from breaking file structure
- Both ordered ASC (oldest first) for natural reading flow
- Author field: agent name, "System", or "User"

#### Task Output File

Location: `/tmp/malamar_output_{random_nanoid}.json` (pre-created, unique per execution)

```json
{
  "actions": [
    { "type": "comment", "content": "## Implementation Complete\n\nI've added the feature..." },
    { "type": "change_status", "status": "in_review" }
  ]
}
```

### Chat Context Passing

#### Chat Input File

Location: `/tmp/malamar_chat_{chat_id}.md` (overwritten for each message processing)

```markdown
# Malamar Chat Context

You are being invoked by Malamar's chat feature.
{agent instruction here}

## Chat Metadata

- Chat ID: abc123
- Workspace: My Project
- Agent: Malamar (or agent name)

## Conversation History

​```json
{"role": "user", "content": "Help me create a reviewer agent", "created_at": "2025-01-17T10:00:00Z"}
{"role": "agent", "content": "Sure! What kind of work will this reviewer focus on?", "created_at": "2025-01-17T10:00:30Z"}
{"role": "user", "content": "Code review for TypeScript", "created_at": "2025-01-17T10:01:00Z"}
{"role": "system", "content": "User has uploaded /tmp/malamar_chat_abc123_attachments/style-guide.md", "created_at": "2025-01-17T10:01:30Z"}
​```

## Additional Context

For workspace state and settings, read: /tmp/malamar_chat_abc123_context.md

# Output Instruction

Write your response as JSON to: /tmp/malamar_chat_abc123_output.json
```

**Format Notes:**
- Conversation history in JSONL format (one JSON per line) inside code block
- All messages included (user, agent, and system) in ASC order
- System messages contain error info, file attachment notifications, and agent switch notifications (e.g., "Changed agent from Planner to Malamar")
- Only user messages trigger CLI responses - system messages NEVER trigger the agent to respond

#### Chat Context File

Location: `/tmp/malamar_chat_{chat_id}_context.md` (separate file for workspace state)

This file is referenced in the input file but agents read it only when needed. Contains:
- Workspace settings (title, description, working directory, cleanup settings, notifications)
- All agents with their IDs and instructions (ordered), e.g., "Planner (id: 123456abcdef)"
- Global settings (available CLIs and their health status)
- Mailgun configuration status (configured/not configured) - enables agents to warn users about missing setup without exposing credentials

Task summaries are NOT included.

**Note on Agent IDs:** Agent IDs are included in chat context files because agents can perform structured actions requiring IDs. Task input files use names-only since task agents communicate through comments (human-readable) and have no actions requiring IDs.

#### Chat Output File

Location: `/tmp/malamar_chat_output_{random_nanoid}.json` (fresh file each processing)

```json
{
  "message": "I've created a Reviewer agent for TypeScript code review.",
  "actions": [
    { "type": "create_agent", "name": "Reviewer", "instruction": "...", "cli_type": "claude", "order": 3 }
  ]
}
```

**Format Notes:**
- `message` field is optional but encouraged
- `actions` array is optional; if present, each action is executed immediately
- Uses random nanoid (not chat_id) to prevent the dangerous scenario where a failed CLI run leaves the previous output file intact, causing Malamar to parse stale data and potentially duplicate messages or actions

**File Pattern Summary:**
- Chat input: `/tmp/malamar_chat_{chat_id}.md` (reused per chat)
- Chat output: `/tmp/malamar_chat_output_{random_nanoid}.json` (fresh each processing)

### Chat Working Directory

The chat CLI is invoked with a working directory that respects the workspace setting:

- **Temp Folder mode**: `/tmp/malamar_chat_{chat_id}` (created per chat)
- **Static Directory mode**: The path configured in workspace settings

This allows chat agents to have full access to the workspace's working environment (e.g., browse repository files, run commands, make changes).

### Task Working Directory Management

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

**Timeout:** There is no timeout consideration for chat processing. Cancellation is purely user-initiated via the stop button in the chat UI.

### Chat Attachments

Location: `/tmp/malamar_chat_{chat_id}_attachments/{filename}`

- Files are stored here when users upload them
- A system message is added to the chat noting the file path
- Duplicate filenames overwrite existing files
- No file size or type restrictions
- CLIs that support images/binaries can use them; others treat as text

### Chat Title Behavior

- Default title: "Untitled chat"
- `rename_chat` action available to agents on first response only
- After first agent response, `rename_chat` action becomes unavailable
- User can edit title anytime via UI

**Enforcing First Response Only:**

When processing chat output actions, before executing `rename_chat`:
1. Query `SELECT COUNT(*) FROM chat_messages WHERE chat_id = ? AND role = 'agent'`
2. If count > 0 (meaning there's already an agent response), ignore the `rename_chat` action silently
3. If count = 0, execute the rename

Silent ignore (not an error) because agents may mistakenly try later - no harm in ignoring.

### Chat Queue Processing

Separate chat processor with immediate pickup:

- Maintain a separate polling loop for chat queue (same interval as task runner)
- When user sends a message: create `chat_queue` item with status `queued`, return API response immediately
- Chat processor picks up `queued` chat items
- No "one per workspace" limit - multiple chats can process concurrently
- Track active chat subprocesses in separate `Map<string, Subprocess>` keyed by chat_id (for cancellation)
- On completion: parse output, execute actions, add agent message to chat, emit SSE events
- On error: add system message to chat, mark queue item failed

Simpler than task processing (no multi-agent loop), but uses same subprocess and error handling patterns.

### Action Execution

When an agent returns multiple actions, execute all and collect errors:

- Execute actions in array order
- Each action executes independently (failure of one doesn't stop others)
- Collect all errors and add a single system message summarizing failures
- Example: If `create_agent` fails due to duplicate name but `rename_chat` succeeds, system message says "Action failed: create_agent - Agent name 'Reviewer' already exists"
- The agent's conversational `message` is always added to chat (regardless of action success/failure)

### Chat CLI Override

- Per-chat setting stored in `chats.cli_type` column
- When set, overrides the agent's default CLI for all messages in that chat
- For Malamar Agent (or when no override and no agent CLI), use first healthy CLI in this order:
  1. Claude Code
  2. Codex CLI
  3. Gemini CLI
  4. OpenCode

### Chat Agent Switching

When user switches agents mid-chat:
- Full conversation history is preserved
- System message added: "Switched from [Agent A] to [Agent B]"
- New agent sees entire conversation and continues with full context

---

## API & Real-Time

### HTTP Framework

**Hono** - lightweight, fast web framework designed for edge/Bun environments:

- Simple routing: `app.get('/api/workspaces', handler)`
- Middleware support for error handling, logging
- TypeScript-first with good type inference
- Works with Bun's native APIs (including SSE via native Response)
- Popular choice for Bun projects, well-documented

### Request Validation

Zod schemas with Hono integration:

- Define Zod schemas for each request body type
- Use `@hono/zod-validator` middleware for automatic validation
- Schemas double as TypeScript types (single source of truth)
- Example: `CreateWorkspaceSchema = z.object({ title: z.string().min(1), description: z.string().optional() })`

### Error Response Format

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

### Pagination

No pagination for v1. Return all items in list endpoints. Reasons:

- Single-user app with limited data volume
- Comments/logs per task are typically manageable (tens, not thousands)
- Simplifies frontend implementation

Apply same principle to all list endpoints: workspaces, agents, tasks, comments, logs, chats, messages.

### REST Endpoints

#### Workspaces
- `GET /api/workspaces` - List all workspaces (supports `?q=` for title search)
- `POST /api/workspaces` - Create workspace
- `GET /api/workspaces/:id` - Get workspace details
- `PUT /api/workspaces/:id` - Update workspace
- `DELETE /api/workspaces/:id` - Delete workspace

#### Agents
- `GET /api/workspaces/:id/agents` - List agents in workspace
- `POST /api/workspaces/:id/agents` - Create agent
- `PUT /api/agents/:id` - Update agent
- `DELETE /api/agents/:id` - Delete agent
- `PUT /api/workspaces/:id/agents/reorder` - Reorder agents

#### Tasks
- `GET /api/workspaces/:id/tasks` - List tasks in workspace
- `POST /api/workspaces/:id/tasks` - Create task
- `GET /api/tasks/:id` - Get task details
- `PUT /api/tasks/:id` - Update task
- `DELETE /api/tasks/:id` - Delete task
- `POST /api/tasks/:id/prioritize` - Prioritize task
- `POST /api/tasks/:id/cancel` - Cancel running loop
- `DELETE /api/workspaces/:id/tasks/done` - Delete all done tasks

#### Comments
- `GET /api/tasks/:id/comments` - List comments
- `POST /api/tasks/:id/comments` - Add comment

#### Activity Logs
- `GET /api/tasks/:id/logs` - List activity logs

#### Chats
- `GET /api/workspaces/:id/chats` - List chats in workspace (supports `?q=` for title search)
- `POST /api/workspaces/:id/chats` - Create chat (specify agent_id or null for Malamar)
- `GET /api/chats/:id` - Get chat details with messages
- `PUT /api/chats/:id` - Update chat (title, agent_id, cli_type)
- `DELETE /api/chats/:id` - Delete chat
- `POST /api/chats/:id/messages` - Send a message (triggers processing)
- `POST /api/chats/:id/cancel` - Cancel processing (kills CLI subprocess)
- `POST /api/chats/:id/attachments` - Upload file attachment

#### Settings
- `GET /api/settings` - Get all settings
- `PUT /api/settings` - Update settings
- `POST /api/settings/test-email` - Send test email

#### Health
- `GET /api/health` - Overall health status
- `GET /api/health/cli` - CLI availability status
- `POST /api/health/cli/refresh` - Manually trigger CLI re-detection

### SSE Events

`GET /api/events` - Server-Sent Events stream

**Implementation Pattern:**

EventEmitter + connection registry:

- Create a singleton `EventEmitter` instance for pub/sub
- Maintain a `Set<Response>` of active SSE connections
- On new SSE request: add response to Set, send initial `:ok` comment, set up disconnect cleanup
- When emitting events: `eventEmitter.emit(eventType, payload)` triggers a handler that iterates the Set and writes to each connection
- On client disconnect: remove from Set
- Event format per SSE spec: `event: {type}\ndata: {json}\n\n`
- Include `retry: 3000` on connection start for auto-reconnect hint

**Events:**
```
event: task.status_changed
data: {"task_id": "xxx", "task_summary": "...", "old_status": "todo", "new_status": "in_progress", "workspace_id": "yyy"}

event: task.comment_added
data: {"task_id": "xxx", "task_summary": "...", "author_name": "Planner", "workspace_id": "yyy"}

event: task.error_occurred
data: {"task_id": "xxx", "task_summary": "...", "error_message": "CLI exited with code 1", "workspace_id": "yyy"}

event: agent.execution_started
data: {"task_id": "xxx", "task_summary": "...", "agent_name": "Implementer", "workspace_id": "yyy"}

event: agent.execution_finished
data: {"task_id": "xxx", "task_summary": "...", "agent_name": "Implementer", "workspace_id": "yyy"}

event: chat.message_added
data: {"chat_id": "xxx", "chat_title": "...", "author_type": "user|agent|system", "workspace_id": "yyy"}

event: chat.processing_started
data: {"chat_id": "xxx", "chat_title": "...", "agent_name": "Malamar", "workspace_id": "yyy"}

event: chat.processing_finished
data: {"chat_id": "xxx", "chat_title": "...", "agent_name": "Malamar", "workspace_id": "yyy"}
```

**Broadcasting Scope:** SSE events are broadcast to all connected clients without workspace scoping. Client-side filtering handles noise if needed. Workspace-scoped broadcasting will be revisited when adding multi-user authentication.

---

## User Interface Implementation

### Page Structure

| Page | Purpose |
|------|---------|
| **Workspace List** | Home page showing all workspaces as cards with activity summaries |
| **Workspace Detail** | Tabs for Tasks (Kanban), Chat, and Agents |
| **Workspace Settings** | Configure working directory, cleanup, notifications |
| **Task Detail** | View/edit task, comments, activity logs, take actions |
| **Chat Detail** | Conversation with an agent, file attachments |
| **Global Settings** | CLI configuration, Mailgun setup |

### Task Kanban Board

Tasks are displayed in a Kanban board with four columns: Todo, In Progress, In Review, Done.
- Tasks are ordered by most recently updated
- Clicking a task opens the task detail popup
- No drag-and-drop between columns (status changes happen in task detail)

### Real-Time Updates

- React Query with conservative caching (30s stale time) since SSE pushes updates
- SSE events trigger `queryClient.invalidateQueries()` for affected resources
- Refetch on window focus for returning users
- SSE events trigger toast notifications for important events

### Theme Support

- Default: Follow system preference (`prefers-color-scheme`)
- Toggle: Theme switcher in global settings or header
- Preference stored in localStorage
- Uses shadcn/ui's built-in dark mode support

### Accessibility

shadcn/ui components are built on Radix primitives, which handle ARIA attributes, focus management, and keyboard navigation. The codebase follows baseline practices:

- Semantic HTML elements (`button`, `nav`, `main`, `h1-h6`)
- Skip-to-content link in the root layout
- Form error messages associated via `aria-describedby`
- Color contrast meets WCAG AA (via shadcn/ui theme)

### Optimistic Updates

Some actions update the UI immediately before server confirmation for better perceived responsiveness:

**Optimistic (instant UI feedback):**
- Adding a comment to a task
- Sending a chat message
- Toggling task priority
- Theme switching

**Wait for server (show loading state):**
- Creating/deleting workspaces, tasks, agents, or chats
- Status changes (triggers backend logic)
- Agent reordering
- Form submissions with validation

### Search Functionality

Basic text search is available on:
- **Workspace list**: Search by workspace title
- **Chat list**: Search by chat title

No search on the task Kanban board (columns already filter by status).

### Empty States

Each empty list displays a contextual message with a call-to-action:

| Empty State | Display |
|-------------|---------|
| Workspace list | "No workspaces yet" + "Create Workspace" button |
| Tasks (whole board) | "No tasks yet" + "Create Task" button |
| Tasks (single column) | Empty column, no message |
| Chats | "No conversations yet" + "Start a chat" dropdown |
| Agents | Warning banner + prompt to chat with Malamar agent |

### Form Validation

**Required fields:**
- Workspace title
- Agent name (must be unique within workspace)
- Agent instruction
- Task summary
- Comment content (non-empty)
- Chat message (non-empty)

**Optional fields:**
- Workspace instruction
- Task description
- Directory path (static mode) - show warning if path doesn't exist but allow saving

**No max lengths** - UI truncates long text in lists/cards with ellipsis, shows full text in detail views.

### Delete Confirmations

| Action | Confirmation Pattern |
|--------|---------------------|
| Delete Workspace | Type workspace name to confirm |
| Delete All Done Tasks | Type workspace name to confirm |
| Delete Individual Task | Simple confirm dialog |
| Delete Individual Agent | Simple confirm dialog |
| Delete Individual Chat | Simple confirm dialog |

### Task Kanban Cards

Each task card displays:
- Task summary (truncated with ellipsis if long)
- Time since last update (e.g., "5 min ago", "2 days ago")
- Comment count badge (if > 0)
- Priority badge (if prioritized)
- Processing indicator (pulsing border or spinner when agent is working)

### Task Prioritization UI

- **Button location**: Task detail view, alongside other action buttons
- **Visual indicator**: Priority badge/icon on Kanban card
- **Toggle behavior**: Button shows "Prioritize" or "Remove Priority" based on current state

### Agent Ordering UI

- **New agents**: Append to end (highest order + 1)
- **Display**: Agents listed with position numbers (1, 2, 3, 4)
- **Reorder**: Drag-and-drop on desktop, up/down arrows on mobile
- **Save**: Changes save immediately on drop

### Default CLI for New Agents

Pre-select the first healthy CLI in priority order:
1. Claude Code
2. Codex CLI
3. Gemini CLI
4. OpenCode

If none healthy, pre-select Claude Code (user sees unhealthy indicator).

### Toast Notifications

- **Duration**: 5 seconds auto-dismiss
- **Stacking**: Max 3 visible, older dismissed when new arrive
- **Position**: Bottom-right
- **Interaction**: Clickable (navigates to relevant item), X button to dismiss
- **Types**: Success (green), Error (red), Info (neutral)

### Date/Time Formatting

| Age | Format |
|-----|--------|
| < 1 hour | "5 min ago", "just now" |
| < 24 hours | "3 hours ago" |
| < 7 days | "2 days ago" |
| ≥ 7 days (current year) | "Jan 15" |
| ≥ 7 days (past year) | "Jan 15, 2025" |

- Hover tooltip shows full absolute timestamp
- Uses browser locale for formatting
- Times displayed in user's local timezone

### CLI Health Warnings

When CLIs are unhealthy, display warnings:

| Scenario | Warning |
|----------|---------|
| Zero CLIs healthy | Persistent warning in header: "No CLIs available - configure in settings" |
| Agent's CLI unhealthy | Warning banner on workspace page |
| Chat with no healthy CLI | Error in chat UI preventing send: "No CLI available" |
| CLI dropdown | Visual indicator (red dot) for unhealthy CLIs |

### Single Executable Distribution

**Preferred approach:** Bundle frontend build output directly into Bun binary using `bun build --compile`, serving embedded static assets if supported.

**Fallback approach:**
1. Compress UI dist into binary
2. On startup, decompress to `~/.malamar/ui/`
3. Serve static files from that location
4. Overwrite on each startup to ensure UI matches binary version

### Static File Serving

Hono static middleware with embedded files:

- At build time: embed frontend `dist/` files into the binary (Bun supports this)
- At runtime: serve embedded files for non-API routes
- Route priority:
  1. `/api/*` → API handlers
  2. Known static extensions (`.js`, `.css`, `.png`, etc.) → serve from embedded assets
  3. All other routes → serve `index.html` (SPA fallback for client-side routing)
- Set appropriate `Content-Type` headers based on file extension
- Set cache headers for assets (e.g., `Cache-Control: public, max-age=31536000` for hashed filenames)

### CORS Handling

No CORS handling needed. The frontend uses Vite's proxy feature during development to forward `/api/*` requests to `localhost:3456`. The frontend always calls `/api/...` relative URLs, making behavior consistent between development and production.

### File Attachments

Attachments are only for CLI agents to read, not displayed in the UI. No API endpoint for serving attachment files.

---

## Background Jobs

Simple setInterval with startup trigger for all scheduled jobs:

- Store last run timestamp in memory (not persisted - if Malamar restarts, jobs run again which is fine)
- Jobs are async functions that don't block the event loop

### Runner Job

- **Type**: Continuous loop
- **Interval**: Configurable via `MALAMAR_RUNNER_POLL_INTERVAL` (default: 1000ms)
- **Concurrency**: One worker per workspace, unlimited workspaces

### Cleanup Job

- **Type**: Scheduled daily
- **Implementation**: `setInterval(runCleanup, 24 * 60 * 60 * 1000)` + run once on startup
- **Task queue cleanup**: Delete completed/failed task queue items > 7 days old
- **Chat queue cleanup**: Delete completed/failed chat queue items > 7 days old
- **Task cleanup**: Delete done tasks exceeding workspace retention period

### CLI Health Check Job

- **Type**: Scheduled
- **Interval**: Every 5 minutes
- **Implementation**: `setInterval(checkCliHealth, 5 * 60 * 1000)` + run once on startup
- **Action**: For each supported CLI:
  1. Search for binary in PATH (or custom path from settings)
  2. Run CLI with minimal prompt (e.g., `claude --prompt "Reply OK"` or equivalent per CLI)
  3. Health check passes if:
     - Process exits with code 0
     - stdout is non-empty (any response, doesn't need to match "OK")
  4. Health check fails if:
     - Binary not found → error: "Binary not found in PATH"
     - Non-zero exit code → error: "CLI exited with code {code}"
     - Empty output → error: "CLI returned empty response"
     - Timeout (30 seconds) → error: "CLI health check timed out"
  5. Store status and error message in memory
  6. Expose via `GET /api/health/cli` endpoint

**Manual Refresh:** Users can trigger immediate CLI re-detection via the "Refresh CLI Status" button on the CLI Settings page (calls `POST /api/health/cli/refresh`). This gives immediate feedback after installing new CLIs, while the periodic check handles background changes.

---

## Configuration Reference

Configuration via environment variables (priority) or CLI flags.

**Loading Priority** (later wins):
1. Hardcoded defaults (e.g., port 3456)
2. CLI flags (`--port 8080`)
3. Environment variables (`MALAMAR_PORT=9000`)

Parse CLI flags first using Bun's `process.argv` or a minimal parser. Then check for corresponding env vars and override if present. Example: `--port 8080` + `MALAMAR_PORT=9000` → uses 9000.

Expose final config via `malamar config` command.

### Server

| Setting | Env Var | CLI Flag | Default |
|---------|---------|----------|---------|
| Bind address | `MALAMAR_HOST` | `--host` | `127.0.0.1` |
| Server port | `MALAMAR_PORT` | `--port` | `3456` |

### Data & Storage

| Setting | Env Var | CLI Flag | Default |
|---------|---------|----------|---------|
| Data directory | `MALAMAR_DATA_DIR` | `--data-dir` | `~/.malamar` |

### Logging

| Setting | Env Var | CLI Flag | Default |
|---------|---------|----------|---------|
| Log verbosity | `MALAMAR_LOG_LEVEL` | `--log-level` | `info` |
| Output format | `MALAMAR_LOG_FORMAT` | `--log-format` | `text` |

Log levels: `debug`, `info`, `warn`, `error` (each includes higher levels)

Log formats: `text`, `json`

**Implementation:**

Simple custom logger module:
- Text format: `[2025-01-18T10:30:00Z] [INFO] Message here { context }`
- JSON format: `{"timestamp":"2025-01-18T10:30:00Z","level":"info","message":"Message here","context":{}}`
- API: `logger.info("message", { optional: "context" })`, `logger.error("message", { error })`
- Write to stdout (text/json based on config)
- No external logging library needed for this scope

### Runner

| Setting | Env Var | CLI Flag | Default |
|---------|---------|----------|---------|
| Poll interval (ms) | `MALAMAR_RUNNER_POLL_INTERVAL` | `--runner-poll-interval` | `1000` |

### Temp Files

| Setting | Env Var | CLI Flag | Default |
|---------|---------|----------|---------|
| Temp directory | `MALAMAR_TEMP_DIR` | `--temp-dir` | System `/tmp` |

**Cleanup Policy:** Malamar does not proactively clean up its own temp files. OS-dependent `/tmp` cleanup is acceptable. Users needing control can use `MALAMAR_TEMP_DIR` to manage their own temp directory.

---

## Operations

### CLI Commands

| Command | Description |
|---------|-------------|
| `malamar` | Start the server (default) |
| `malamar version` | Show version info |
| `malamar help` | Show help/usage |
| `malamar doctor` | Check system health (CLIs, database, config) |
| `malamar config` | Show current configuration |

**`malamar doctor` Output:**

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

### First Startup Detection

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

### First Startup

On first launch (empty database):
- Auto-create sample workspace: "Sample: Code Assistant"
- 4 agents: Planner, Implementer, Reviewer, Approver
- Each with generic but useful instructions demonstrating the workflow
- No sample tasks - user creates their first task
- Sample workspace can be deleted when no longer needed

### Concurrent Access

- No locking mechanism
- Multiple tabs can edit - last save wins
- SSE and polling keep all tabs synchronized
- If user comments while agent is processing, comment is added and agent continues (sees it on next loop iteration)
- Simple approach matching single-user assumption

### Mailgun Integration

Email notifications are sent via fire-and-forget async calls. No separate background job - triggered inline but non-blocking.

**Failure Handling:** Failures are silently logged to application error log. No user-facing alert for notification failures.

**Test Email Endpoint (`POST /api/settings/test-email`):**

- Request body: empty (uses saved Mailgun settings)
- Validate settings exist (API key, domain, from/to emails) → 400 if missing
- Send email with:
  - Subject: "Malamar Test Email"
  - Body: "This is a test email from Malamar. If you received this, your email notifications are configured correctly."
- Wait for Mailgun API response (don't fire-and-forget for test)
- On success: return 200 `{ "success": true }`
- On Mailgun error: return 400 `{ "error": { "code": "EMAIL_FAILED", "message": "Mailgun error: {details}" } }`

### Domain Agnostic Design

Malamar is purely an orchestration layer. It has no opinions on what agents do - all domain-specific behavior is defined in agent instructions. Use cases include:

- Software development (coding, reviewing, pushing)
- Writing blog posts
- Browser automation via CLI
- System management
- HR spreadsheet work
- Any task achievable via AI CLIs

Malamar only cares about the agent response format so the runner can understand the outcome.

---

## Testing

### Test Types

| Type | Location | What it Tests | Server Running? |
|------|----------|---------------|-----------------|
| **Unit** | `src/**/*.test.ts` | Individual functions, mocked dependencies | No |
| **Integration** | `tests/` | Service layers with real database | No |
| **E2E** | `e2e/` | HTTP requests against running server | Yes |

Unit tests are co-located with source files for discoverability. Integration and E2E tests have separate directories.

### Test Isolation

E2E tests use environment variables for isolation:
- `MALAMAR_DATA_DIR=/tmp/malamar-test` - Isolated data directory
- `MALAMAR_PORT=3457` - Non-default port

Each E2E test file manages its own server lifecycle (start before tests, stop after). Tests clean up before and after to ensure a clean slate.

---

## Development Tooling

| Tool | Purpose |
|------|---------|
| **Bun** | Runtime, package manager, test runner, bundler |
| **TypeScript** | Type safety (run directly via Bun on backend) |
| **Prettier** | Code formatting |
| **ESLint** | Linting with `eslint-plugin-simple-import-sort` for import organization |
| **husky + lint-staged** | Pre-commit hooks to run ESLint and Prettier on staged files |
