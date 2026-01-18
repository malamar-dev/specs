# Technical Design: Runtime

This document covers the core runtime mechanics of Malamar: the runner, queue processing, CLI adapters, and context passing.

For product specifications (what and why), see [SPECS.md](./SPECS.md).

---

## Table of Contents

- [Runner](#runner)
- [Task Queue Processing](#task-queue-processing)
- [Chat Queue Processing](#chat-queue-processing)
- [CLI Adapters](#cli-adapters)
- [Context Passing](#context-passing)
- [Background Jobs](#background-jobs)

---

## Runner

The runner is the main background job that processes tasks. It runs as a continuous loop with configurable polling interval (default: 1000ms, configurable via `MALAMAR_RUNNER_POLL_INTERVAL`).

### Concurrent Workers

- Each workspace has its own worker that handles queue pickup and agent loops
- Multiple workspaces can process tasks simultaneously
- Within a workspace, only one task is processed at a time
- No artificial limit on concurrent workspace workers

### Loop Flow

```
1. Pick queue item (see Queue Pickup Algorithm)
2. Move task to "In Progress" (if in "Todo")
3. Auto-demote other "In Progress" tasks to "Todo"
4. For each agent (by order):
   a. Fetch agent data (just-in-time)
   b. Create input file
   c. Invoke CLI
   d. Read output file
   e. Execute actions (skip/comment/change_status)
   f. If change_status to "In Review", stop loop
5. After all agents:
   - If any comment added: retrigger loop from first agent
   - If all skipped: move task to "In Review"
```

### Agent Execution (Just-in-Time Snapshot)

Before executing each agent:
1. Query for the next agent by finding the smallest `order` value greater than current
2. Fetch agent data (instruction, name, CLI) right before execution
3. Changes to agents take effect immediately for subsequent agents

**Implications:**
- If an agent is deleted mid-loop, it's simply not found and skipped
- If a new agent is inserted with an `order` between current and next, it will be picked up
- Changes to an agent's instruction don't affect an already-running execution

**Mid-Flight Setting Changes:** If workspace settings (like working directory) are changed while a task is actively being processed, the currently running agent continues with the old settings. Subsequent agents in the loop pick up the new settings.

### Error Handling

The following errors are handled uniformly:
- **CLI execution failure**: Process crashes or returns non-zero exit code
- **CLI timeout**: Process takes too long (handled by OS)
- **Malformed output**: Output file is empty, missing, contains invalid JSON, or doesn't match expected schema

When any error occurs:
1. Write System comment with detailed error information (e.g., "Output file was empty", "Invalid JSON: unexpected token at position 42", "CLI exited with code 1")
2. Stop current loop
3. System comment emits task event → new queue item
4. Task stays in current status (no move to "In Review")
5. Natural retry via queue (no explicit backoff)

### Workspace With No Agents

When a workspace has zero agents:
- Tasks immediately move to "In Review" (no agents = all skip)
- Chats can still happen with the Malamar agent to recreate agents

---

## Task Queue Processing

### Queue Pickup Algorithm

Per workspace, the runner picks up queue items in this order:

1. **Status filter first**: Only consider items where task status is "Todo" or "In Progress". Items for tasks in "In Review" or "Done" are ignored.
2. **Priority flag**: Items with `is_priority: true` and status `queued`
3. **Most recently processed**: Find the task with the most recent `completed` or `failed` queue item, pick its `queued` item
4. **LIFO fallback**: Pick the most recently updated `queued` item (by `updated_at`)

**Rationale**: Tackle one task until completion, rather than starting many tasks but finishing none. LIFO ordering allows users to "bump" a task by commenting (which updates the queue item's `updated_at`).

### Queue Behavior

1. When a new task event is emitted, if the queue for that task is empty, a new item is added with status "queued"
2. When the runner picks up a queue item, its status changes to "in_progress"
3. If a new event is emitted while an item is "in_progress", a new item is added with status "queued"
4. If a new event is emitted while there's already both an "in_progress" and a "queued" item, no new item is added - but the existing "queued" item's `updated_at` is refreshed
5. Completed and failed items are kept for pickup priority logic and cleaned up after 7 days

### Task Events

A task event is emitted when:
- User creates a new task
- User/agent/system comments on a task
- Task status/properties has been changed

### Priority Override

Users can manually prioritize a task for the next loop:

1. Find or create a queue item for that task
2. Set `is_priority: true` on that queue item
3. Set `is_priority: false` on all other queue items in that workspace
4. The runner will pick up this task next

Only one task can be prioritized at a time per workspace. After processing, the runner returns to normal pickup behavior.

**Behavior with running tasks:** Prioritization does NOT interrupt running tasks. The prioritized task waits until the current loop finishes. If urgent, the user should prioritize AND manually cancel the currently running task.

### Status Transitions

**Runner-triggered:**
1. If task is in "Todo", move it to "In Progress"
2. If task is in "In Progress" but all agents skip, move it to "In Review"
3. If task is in "In Review" but there is a new user comment, move it back to "In Progress" and retrigger the loop
4. If task is in "Done", do nothing

**Auto-demotion:** When the runner picks a task for a new loop, all OTHER "In Progress" tasks in that workspace are automatically moved to "Todo".

**Queue Item Creation:** Moving a task to "In Progress" or "Todo" always creates a queue item if one doesn't already exist.

### Cancel Loop

When a user cancels a running loop:
1. Send SIGTERM to the running CLI subprocess
2. Leave the output file as-is (no cleanup)
3. Add a system comment noting the user canceled the loop
4. Task stays in "In Progress" (does NOT move to "In Review")
5. Runner will re-pick the task following normal queue rules

**Child Processes:** Malamar is only responsible for terminating the CLI process itself. Child processes are the CLI's responsibility.

---

## Chat Queue Processing

### Chat Queue Behavior

- Chat processing is tracked via `chat_queue` table with same states as task queue
- No PID storage needed - the processor holds the subprocess reference in memory
- No timeout consideration - cancellation is purely user-initiated via stop button

### Concurrency

- Chats within one workspace CAN process concurrently (unlike tasks which are one-at-a-time)
- No limit on concurrent chat processing across workspaces
- UI prevents sending multiple messages in the same chat while a response is pending

### Processing Flow

1. User sends a message
2. Malamar collects all messages plus metadata (agent instruction, workspace context)
3. Serialize into a markdown file at `/tmp/malamar_chat_{chat_id}.md`
4. Instruct the CLI to read and respond
5. CLI writes response to `/tmp/malamar_chat_output_{random_nanoid}.json` (fresh file each time)
6. Malamar parses response, executes any actions, displays message

### Working Directory

Chat respects the workspace's working directory setting:
- **Temp Folder mode**: Creates `/tmp/malamar_chat_{chat_id}` as the working directory
- **Static Directory mode**: Uses the directory defined in workspace settings

### Error Handling

Errors are displayed as system messages in the chat (e.g., "Error: CLI exited with code 1"). System messages are visible to users and included in the input file when the conversation continues.

---

## CLI Adapters

Each supported CLI has an adapter with hardcoded configuration.

### Supported CLIs

| CLI | Binary Name |
|-----|-------------|
| Claude Code | `claude` |
| Gemini CLI | `gemini` |
| OpenAI Codex CLI | `codex` |
| OpenCode | `opencode` |

### CLI Invocation

Each CLI is invoked via direct shell command with configured environment variables:

```shell
# Claude Code (supports JSON schema)
claude --dangerously-skip-permissions \
  --json-schema '{"type":"object",...}' \
  --output-format json \
  --prompt "Read the file at /tmp/malamar_task_xxx.md and follow the instruction autonomously."

# Other CLIs (schema in prompt)
gemini --prompt "Read the file at /tmp/malamar_task_xxx.md and follow the instruction autonomously."
```

The runner waits for the subprocess to exit before proceeding.

### Response Format Enforcement

- **CLIs with schema support** (Claude Code): Use native `--json-schema` flag
- **CLIs without schema support**: Embed JSON format instruction in the prompt

### Auto-Detection

On startup and every 5 minutes:
1. Search for binary in PATH (or custom path from settings)
2. Run test prompt: "Respond with OK"
3. Verify successful exit with non-empty output
4. Update in-memory status ("Healthy" or "Unhealthy")

**Manual Refresh:** Users can trigger immediate re-detection via "Refresh CLI Status" button (calls `POST /api/health/cli/refresh`).

### CLI Settings

| Field | Description |
|-------|-------------|
| Binary Path | Custom path override (empty = use PATH) |
| Environment Variables | Key-value pairs to inject when running |
| Detected Version | Read-only, auto-detected |
| Status | "Healthy" or "Unhealthy" |
| Error Message | Details when unhealthy |

**Locked Down Templates:** CLI command templates are intentionally locked down for stability. Users can customize binary paths and environment variables, but not command templates.

---

## Context Passing

When the runner invokes an agent's CLI, it passes context through temporary files.

### Task Input File

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

### Task Output File

Location: `/tmp/malamar_output_{random_nanoid}.json` (pre-created, unique per execution)

```json
{
  "actions": [
    { "type": "comment", "content": "## Implementation Complete\n\nI've added the feature..." },
    { "type": "change_status", "status": "in_review" }
  ]
}
```

**Action Schema:**

| Action | Fields |
|--------|--------|
| `skip` | `{ "type": "skip" }` |
| `comment` | `{ "type": "comment", "content": "<markdown>" }` |
| `change_status` | `{ "type": "change_status", "status": "in_review" }` |

### Chat Input File

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

**Notes:**
- Conversation history in JSONL format, all messages (user, agent, system) in ASC order
- Only user messages trigger CLI responses - system messages NEVER trigger the agent to respond

### Chat Context File

Location: `/tmp/malamar_chat_{chat_id}_context.md` (separate file for workspace state)

Contains:
- Workspace settings (title, description, working directory, cleanup settings, notifications)
- All agents with their IDs and instructions (ordered), e.g., "Planner (id: 123456abcdef)"
- Global settings (available CLIs and their health status)
- Mailgun configuration status (configured/not configured)

**Note on Agent IDs:** Agent IDs are included in chat context because agents can perform structured actions requiring IDs. Task input files use names-only since task agents communicate through comments.

### Chat Output File

Location: `/tmp/malamar_chat_output_{random_nanoid}.json` (fresh file each processing)

```json
{
  "message": "I've created a Reviewer agent for TypeScript code review.",
  "actions": [
    { "type": "create_agent", "name": "Reviewer", "instruction": "...", "cli_type": "claude", "order": 3 }
  ]
}
```

**Notes:**
- `message` field is optional but encouraged
- `actions` array is optional; if present, each action is executed immediately
- Uses random nanoid (not chat_id) to prevent parsing stale data from failed runs

**Chat Action Types - All Agents:**
- `rename_chat`: `{ "type": "rename_chat", "title": "..." }`
- `create_task`: `{ "type": "create_task", "summary": "...", "description": "..." }`

**Chat Action Types - Malamar Agent Only:**
- `create_agent`: `{ "type": "create_agent", "name": "...", "instruction": "...", "cli_type": "...", "order": number }`
- `update_agent`: `{ "type": "update_agent", "agent_id": "...", "name?": "...", "instruction?": "...", "cli_type?": "..." }`
- `delete_agent`: `{ "type": "delete_agent", "agent_id": "..." }`
- `reorder_agents`: `{ "type": "reorder_agents", "agent_ids": ["id1", "id2", "id3"] }`
- `update_workspace`: `{ "type": "update_workspace", "title?": "...", "description?": "...", ... }`

**Note:** `delete_workspace` is intentionally excluded - workspace deletion requires explicit human confirmation.

### Chat Attachments

Location: `/tmp/malamar_chat_{chat_id}_attachments/{filename}`

- Files stored here when users upload them
- System message added noting the file path
- Duplicate filenames overwrite existing files
- Cleanup left for OS `/tmp` cleanup

### File Pattern Summary

| Purpose | Location |
|---------|----------|
| Task input | `/tmp/malamar_task_{task_id}.md` |
| Task output | `/tmp/malamar_output_{random_nanoid}.json` |
| Chat input | `/tmp/malamar_chat_{chat_id}.md` |
| Chat context | `/tmp/malamar_chat_{chat_id}_context.md` |
| Chat output | `/tmp/malamar_chat_output_{random_nanoid}.json` |
| Chat attachments | `/tmp/malamar_chat_{chat_id}_attachments/{filename}` |
| Task working dir (temp mode) | `/tmp/malamar_tasks_{task_id}` |
| Chat working dir (temp mode) | `/tmp/malamar_chat_{chat_id}` |

---

## Background Jobs

### Runner Job

- **Type**: Continuous loop
- **Interval**: Configurable via `MALAMAR_RUNNER_POLL_INTERVAL` (default: 1000ms)
- **Concurrency**: One worker per workspace, unlimited workspaces

### Cleanup Job

- **Type**: Scheduled daily
- **Tasks**:
  - Delete completed/failed task queue items > 7 days old
  - Delete completed/failed chat queue items > 7 days old
  - Delete done tasks exceeding workspace retention period
  - Cascade delete all associated data

### CLI Health Check Job

- **Type**: Scheduled
- **Interval**: Every 5 minutes
- **Action**: Test each CLI, update in-memory status