# Technical Design: Data Model

This document covers the database schema, entity relationships, and data management for Malamar.

For product specifications (what and why), see [SPECS.md](./SPECS.md).

---

## Table of Contents

- [Database](#database)
- [ID Generation](#id-generation)
- [Entity Schemas](#entity-schemas)
- [Cascade Deletes](#cascade-deletes)
- [Global Settings](#global-settings)

---

## Database

Malamar uses SQLite as its database, stored at `~/.malamar/malamar.db`.

**Migrations:** The application runs database migrations automatically on startup. If a migration fails, the application panics to ensure data consistency.

---

## ID Generation

All IDs use nanoid format. The mock user ID is `000000000000000000000` (21 zeros, a valid nanoid).

**Note:** Currently there is only one user with no auth, so a single mock user ID is used everywhere. Multiple users will be supported later.

---

## Entity Schemas

### Workspaces

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

**Notes:**
- `last_activity_at` is updated whenever any tracked event occurs (task created, status changed, comment added, agent execution started/finished, etc.)
- Task cleanup settings are per-workspace with no global default configuration. The 7-day retention is hardcoded as the default for new workspaces.

### Agents

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

**Notes:**
- `order` must be unique within the workspace
- When reordering agents, the client must send all the new valid ordering

### Tasks

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

### Task Comments

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

**Notes:**
- Either `user_id` or `agent_id` is set, or both are NULL (system comment)
- If a comment was made by an agent that has since been deleted, display "(Deleted Agent)" as the author name

### Task Activity Logs

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

**Event Types:**
- `created` - Task created
- `status_changed` - Task status changed (metadata: `{old_status, new_status}`)
- `comment_added` - Comment added
- `agent_started` - Agent execution started (metadata: `{agent_name}`)
- `agent_finished` - Agent execution finished
- `properties_edited` - Task title/description edited
- `loop_canceled` - Loop canceled by user

### Task Event Queue

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

**Notes:**
- `is_priority` is used for user-initiated priority override
- `updated_at` is used for LIFO ordering and can be "bumped" by updating

### Chats

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

**CLI Selection Behavior:**
1. **Malamar agent** (agent_id is NULL): Uses first available CLI in order: Claude Code → Gemini CLI → Codex CLI → OpenCode
2. **Workspace agents**: Uses the CLI configured on the agent
3. **CLI override** (cli_type is not NULL): Overrides both of the above

### Chat Messages

```
chat_messages
├── id: string (nanoid, primary key)
├── chat_id: string (foreign key)
├── role: enum ("user" | "agent" | "system")
├── message: string (the conversational text)
├── actions: json? (null for user/system, optional array for agent)
├── created_at: datetime
```

**System Messages:**
- Error information (e.g., "Error: CLI exited with code 1")
- File attachment notifications (e.g., "User has uploaded /tmp/malamar_chat_{chat_id}_attachments/{filename}")
- Agent switch notifications (e.g., "Changed agent from Planner to Malamar")

**Note:** Chats do not have a separate activity log table. The message history (including system messages) serves as the audit trail.

### Chat Queue

```
chat_queue
├── id: string (nanoid, primary key)
├── chat_id: string (foreign key)
├── workspace_id: string (foreign key)
├── status: enum ("queued" | "in_progress" | "completed" | "failed")
├── created_at: datetime
└── updated_at: datetime
```

---

## Cascade Deletes

When entities are deleted, associated data is cascade deleted:

| Entity Deleted | Cascaded Deletions |
|----------------|-------------------|
| **Workspace** | All agents, tasks, comments, activity logs, task queue items, chats, chat messages, and chat queue items |
| **Task** | All comments, activity logs, and queue items |
| **Chat** | All chat messages and chat queue items (attachment directory left for OS cleanup) |
| **Agent** | Comments and chats retain `agent_id` for reference; comments display "(Deleted Agent)", chats are switched to Malamar agent |

---

## Global Settings

Global settings are stored in a key-value table:

```
settings
├── key: string (primary key)
└── value: json
```

**Settings Keys:**

| Key | Description |
|-----|-------------|
| `mailgun_api_key` | Mailgun API key |
| `mailgun_domain` | Mailgun domain |
| `mailgun_from_email` | Sender email address |
| `mailgun_to_email` | Recipient email address |
| `notify_on_error` | Global default for error notifications |
| `notify_on_in_review` | Global default for "in review" notifications |
| `cli_settings` | JSON object with per-CLI configuration |

**CLI Settings Structure:**

```json
{
  "claude": {
    "binary_path": "/custom/path/claude",
    "env_vars": {
      "ANTHROPIC_API_KEY": "..."
    }
  },
  "gemini": {
    "binary_path": "",
    "env_vars": {}
  }
}
```