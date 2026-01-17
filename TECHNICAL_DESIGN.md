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

- **Workspace deleted**: All agents, tasks, comments, activity logs, and queue items
- **Task deleted**: All comments, activity logs, and queue items
- **Agent deleted**: Comments retain `agent_id` for reference, display shows "(Deleted Agent)"

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

---

## External Integrations

### CLI Adapter Implementation

Each supported CLI has an adapter with:
- Command template
- Supported flags
- Response format enforcement method

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

#### Settings
- `GET /api/settings` - Get all settings
- `PUT /api/settings` - Update settings
- `POST /api/settings/test-email` - Send test email

#### Health
- `GET /api/health` - Overall health status
- `GET /api/health/cli` - CLI availability status

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

event: agent.execution_started
data: {"task_id": "xxx", "task_summary": "...", "agent_name": "Implementer", "workspace_id": "yyy"}

event: agent.execution_finished
data: {"task_id": "xxx", "task_summary": "...", "agent_name": "Implementer", "workspace_id": "yyy"}
```

Backend uses in-process pub/sub event emitter to fan out to connected clients.

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
- **Queue cleanup**: Delete completed/failed items > 7 days old
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
