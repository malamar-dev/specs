# Malamar Technical Design

This document covers the technical implementation details for Malamar. For product specifications (what and why), see [SPECS.md](./SPECS.md).

## Table of Contents

- [Data Model](#data-model)
- [Core Runtime](#core-runtime)
- [External Integrations](#external-integrations)
- [User Interface](#user-interface)
- [Operations](#operations)

---

## Data Model

### Database

Malamar uses SQLite as its database, stored at `~/.malamar/malamar.db`. The application runs database migrations automatically on startup. If a migration fails, the application panics to ensure data consistency.

### ID Generation

All IDs use nanoid format. The mock user ID is `000000000000000000000` (21 zeros, a valid nanoid).

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

## Core Runtime

### Runner

The runner is the main background job that processes tasks. It runs as a continuous loop with configurable polling interval (default: 1000ms).

#### Concurrent Workers

- Each workspace has its own worker that handles queue pickup and agent loops
- Multiple workspaces can process tasks simultaneously
- Within a workspace, only one task is processed at a time
- No artificial limit on concurrent workspace workers

#### Queue Pickup Algorithm

Per workspace, the runner picks up queue items in this order:

1. **Status filter**: Only consider items where task status is "Todo" or "In Progress"
2. **Priority flag**: Items with `is_priority: true` and status `queued`
3. **Most recently processed**: Find the task with the most recent `completed` or `failed` queue item, pick its `queued` item
4. **LIFO fallback**: Pick the most recently updated `queued` item

#### Agent Execution (Just-in-Time Snapshot)

Before executing each agent:
1. Query for the next agent by finding the smallest `order` value greater than current
2. Fetch agent data (instruction, name, CLI) right before execution
3. Changes to agents take effect immediately for subsequent agents

**Mid-Flight Changes:** If workspace settings (like working directory) are changed while a task is actively being processed, the currently running agent continues with the old settings. Subsequent agents in the loop pick up the new settings. This is an accepted risk that applies to both Malamar agent actions and manual user changes.

#### Loop Flow

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

#### Error Handling

On CLI execution failure, timeout, or malformed output:
1. Write System comment with error details
2. Stop current loop
3. System comment emits task event → new queue item
4. Task stays in current status (no move to "In Review")
5. Natural retry via queue (no explicit backoff)

### Cleanup Job

A single daily background job handling:

**Queue Cleanup:**
- Delete `completed` and `failed` queue items older than 7 days
- Leave `queued` and `in_progress` items untouched

**Done Task Cleanup:**
- Delete tasks in "Done" status exceeding workspace retention period
- Cascade delete all associated data

### CLI Health Check

Runs every 5 minutes:
1. For each supported CLI, search for binary in PATH (or custom path)
2. Run test prompt: "Respond with OK"
3. Verify successful exit with non-empty output
4. Update in-memory status ("Healthy" or "Unhealthy")

**Manual Refresh:** Users can trigger immediate CLI re-detection via the "Refresh CLI Status" button on the CLI Settings page (calls `POST /api/health/cli/refresh`). This gives immediate feedback after installing new CLIs, while the periodic check handles background changes.

---

## External Integrations

### CLI Adapter Implementation

Each supported CLI has an adapter with:
- Command template
- Supported flags
- Response format enforcement method

**Locked Down Templates:** CLI command templates are intentionally locked down for stability. Users can customize binary paths and environment variables via CLI Settings, but not the command templates themselves. CLI adapters are hardcoded.

#### CLI Invocation

```shell
# Claude Code (supports JSON schema)
claude --dangerously-skip-permissions \
  --json-schema '{"type":"object",...}' \
  --output-format json \
  --prompt "Read the file at /tmp/malamar_task_xxx.md and follow the instruction autonomously."

# Other CLIs (schema in prompt)
gemini --prompt "Read the file at /tmp/malamar_task_xxx.md and follow the instruction autonomously."
```

Environment variables from CLI settings are injected into the subprocess environment.

#### Response Format Enforcement

- **CLIs with schema support** (Claude Code): Use native `--json-schema` flag
- **CLIs without schema support**: Embed JSON format instruction in the prompt

### Context Passing

#### Input File

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

#### Output File

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

#### Chat Working Directory

The chat CLI is invoked with a working directory that respects the workspace setting:

- **Temp Folder mode**: `/tmp/malamar_chat_{chat_id}` (created per chat)
- **Static Directory mode**: The path configured in workspace settings

This allows chat agents to have full access to the workspace's working environment (e.g., browse repository files, run commands, make changes).

**Timeout:** There is no timeout consideration for chat processing. Cancellation is purely user-initiated via the stop button in the chat UI.

#### Chat Attachments

Location: `/tmp/malamar_chat_{chat_id}_attachments/{filename}`

Files are stored here when users upload them. A system message is added to the chat noting the file path. Duplicate filenames overwrite existing files.

### Mailgun Integration

Email notifications are sent via fire-and-forget async calls. No separate background job - triggered inline but non-blocking.

**Failure Handling:** Failures are silently logged to application error log. No user-facing alert for notification failures.

---

## User Interface

### Tech Stack

**Backend:**
- TypeScript with Bun runtime
- RESTful API
- Background jobs (runner, cleanup, health check)
- SSE endpoint for real-time updates
- SQLite database

**Frontend:**
- TypeScript with React (via create-vite-app)
- Bun as the build tool/runtime

### API Endpoints

#### Workspaces
- `GET /api/workspaces` - List all workspaces
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
- `GET /api/workspaces/:id/chats` - List chats in workspace
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
- `POST /api/health/cli/refresh` - Manually trigger CLI re-detection (for "Refresh CLI Status" button)

### Real-Time Updates

#### Polling
- Task detail view polls every 3 seconds
- Uses React Query (or similar) for caching and deduplication

#### SSE Endpoint

`GET /api/events` - Server-Sent Events stream

Events:
```
event: task.status_changed
data: {"task_id": "xxx", "task_summary": "...", "old_status": "todo", "new_status": "in_progress", "workspace_id": "yyy"}

event: task.comment_added
data: {"task_id": "xxx", "task_summary": "...", "author_name": "Planner", "workspace_id": "yyy"}

event: task.error_occurred
data: {"task_id": "xxx", "task_summary": "...", "error_message": "CLI exited with code 1", "workspace_id": "yyy"}

event: task.created
data: {"task_id": "xxx", "task_summary": "...", "workspace_id": "yyy"}

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

Backend uses in-process pub/sub event emitter to fan out to connected clients.

**Task Created via Chat:** The `task.created` event is emitted when an agent uses the `create_task` action from chat. The UI shows a toast with a clickable link to the new task.

**Broadcasting Scope:** SSE events are broadcast to all connected clients without workspace scoping. Client-side filtering handles noise if needed. Workspace-scoped broadcasting will be revisited when adding multi-user authentication.

### Single Executable Distribution

**Preferred approach:** Bundle frontend build output directly into Bun binary using `bun build --compile`, serving embedded static assets if supported.

**Fallback approach:**
1. Compress UI dist into binary
2. On startup, decompress to `~/.malamar/ui/`
3. Serve static files from that location
4. Overwrite on each startup to ensure UI matches binary version

---

## Operations

### Configuration

Configuration via environment variables (priority) or CLI flags.

#### Server

| Setting | Env Var | CLI Flag | Default |
|---------|---------|----------|---------|
| Bind address | `MALAMAR_HOST` | `--host` | `127.0.0.1` |
| Server port | `MALAMAR_PORT` | `--port` | `3456` |

#### Data & Storage

| Setting | Env Var | CLI Flag | Default |
|---------|---------|----------|---------|
| Data directory | `MALAMAR_DATA_DIR` | `--data-dir` | `~/.malamar` |

#### Logging

| Setting | Env Var | CLI Flag | Default |
|---------|---------|----------|---------|
| Log verbosity | `MALAMAR_LOG_LEVEL` | `--log-level` | `info` |
| Output format | `MALAMAR_LOG_FORMAT` | `--log-format` | `text` |

Log levels: `debug`, `info`, `warn`, `error`
Log formats: `text`, `json`

#### Runner

| Setting | Env Var | CLI Flag | Default |
|---------|---------|----------|---------|
| Poll interval (ms) | `MALAMAR_RUNNER_POLL_INTERVAL` | `--runner-poll-interval` | `1000` |

#### Temp Files

| Setting | Env Var | CLI Flag | Default |
|---------|---------|----------|---------|
| Temp directory | `MALAMAR_TEMP_DIR` | `--temp-dir` | System `/tmp` |

**Cleanup Policy:** Malamar does not proactively clean up its own temp files. OS-dependent `/tmp` cleanup is acceptable. Users needing control can use `MALAMAR_TEMP_DIR` to manage their own temp directory.

### CLI Commands

| Command | Description |
|---------|-------------|
| `malamar` | Start the server (default) |
| `malamar version` | Show version info |
| `malamar help` | Show help/usage |
| `malamar doctor` | Check system health (CLIs, database, config) |
| `malamar config` | Show current configuration |
| `malamar export` | Export data (details TBD) |
| `malamar import` | Import data (details TBD) |

### Background Jobs

#### Runner Job

- **Type**: Continuous loop
- **Interval**: Configurable via `MALAMAR_RUNNER_POLL_INTERVAL` (default: 1000ms)
- **Concurrency**: One worker per workspace, unlimited workspaces

#### Cleanup Job

- **Type**: Scheduled daily
- **Task queue cleanup**: Delete completed/failed task queue items > 7 days old
- **Chat queue cleanup**: Delete completed/failed chat queue items > 7 days old
- **Task cleanup**: Delete done tasks exceeding workspace retention period

#### CLI Health Check Job

- **Type**: Scheduled
- **Interval**: Every 5 minutes
- **Action**: Test each CLI, update in-memory status

### Domain Agnostic Design

Malamar is purely an orchestration layer. It has no opinions on what agents do - all domain-specific behavior is defined in agent instructions. Use cases include:

- Software development (coding, reviewing, pushing)
- Writing blog posts
- Browser automation via CLI
- System management
- HR spreadsheet work
- Any task achievable via AI CLIs

Malamar only cares about the agent response format so the runner can understand the outcome.
