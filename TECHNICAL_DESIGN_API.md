# Technical Design: API

This document covers the REST API endpoints and real-time events for Malamar.

For product specifications (what and why), see [SPECS.md](./SPECS.md).

---

## Table of Contents

- [REST API Endpoints](#rest-api-endpoints)
- [Real-Time Updates](#real-time-updates)
- [Mailgun Integration](#mailgun-integration)

---

## REST API Endpoints

### Workspaces

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/workspaces` | List all workspaces |
| `POST` | `/api/workspaces` | Create workspace |
| `GET` | `/api/workspaces/:id` | Get workspace details |
| `PUT` | `/api/workspaces/:id` | Update workspace |
| `DELETE` | `/api/workspaces/:id` | Delete workspace |

### Agents

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/workspaces/:id/agents` | List agents in workspace |
| `POST` | `/api/workspaces/:id/agents` | Create agent |
| `PUT` | `/api/agents/:id` | Update agent |
| `DELETE` | `/api/agents/:id` | Delete agent |
| `PUT` | `/api/workspaces/:id/agents/reorder` | Reorder agents |

### Tasks

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/workspaces/:id/tasks` | List tasks in workspace |
| `POST` | `/api/workspaces/:id/tasks` | Create task |
| `GET` | `/api/tasks/:id` | Get task details |
| `PUT` | `/api/tasks/:id` | Update task |
| `DELETE` | `/api/tasks/:id` | Delete task |
| `POST` | `/api/tasks/:id/prioritize` | Prioritize task |
| `POST` | `/api/tasks/:id/cancel` | Cancel running loop |
| `DELETE` | `/api/workspaces/:id/tasks/done` | Delete all done tasks |

### Comments

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/tasks/:id/comments` | List comments |
| `POST` | `/api/tasks/:id/comments` | Add comment |

### Activity Logs

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/tasks/:id/logs` | List activity logs |

### Chats

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/workspaces/:id/chats` | List chats in workspace |
| `POST` | `/api/workspaces/:id/chats` | Create chat (specify agent_id or null for Malamar) |
| `GET` | `/api/chats/:id` | Get chat details with messages |
| `PUT` | `/api/chats/:id` | Update chat (title, agent_id, cli_type) |
| `DELETE` | `/api/chats/:id` | Delete chat |
| `POST` | `/api/chats/:id/messages` | Send a message (triggers processing) |
| `POST` | `/api/chats/:id/cancel` | Cancel processing (kills CLI subprocess) |
| `POST` | `/api/chats/:id/attachments` | Upload file attachment |

### Settings

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/settings` | Get all settings |
| `PUT` | `/api/settings` | Update settings |
| `POST` | `/api/settings/test-email` | Send test email |

### Health

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/health` | Overall health status |
| `GET` | `/api/health/cli` | CLI availability status |
| `POST` | `/api/health/cli/refresh` | Manually trigger CLI re-detection |

---

## Real-Time Updates

Two mechanisms keep the UI in sync:

### Polling

- Task detail view polls every 3 seconds
- Uses React Query (or similar) for caching and deduplication

### Server-Sent Events (SSE)

Endpoint: `GET /api/events`

The backend uses an in-process pub/sub event emitter to fan out events to connected clients.

**Broadcasting Scope:** SSE events are broadcast to all connected clients without workspace scoping. Client-side filtering handles noise if needed. Workspace-scoped broadcasting will be revisited when adding multi-user authentication.

### SSE Event Types

#### Task Events

**task.status_changed**
```json
{
  "task_id": "xxx",
  "task_summary": "...",
  "old_status": "todo",
  "new_status": "in_progress",
  "workspace_id": "yyy"
}
```

**task.comment_added**
```json
{
  "task_id": "xxx",
  "task_summary": "...",
  "author_name": "Planner",
  "workspace_id": "yyy"
}
```

**task.error_occurred**
```json
{
  "task_id": "xxx",
  "task_summary": "...",
  "error_message": "CLI exited with code 1",
  "workspace_id": "yyy"
}
```

**task.created**
```json
{
  "task_id": "xxx",
  "task_summary": "...",
  "workspace_id": "yyy"
}
```
*Note: Emitted when an agent uses the `create_task` action from chat. UI shows a toast with clickable link to the new task.*

#### Agent Events

**agent.execution_started**
```json
{
  "task_id": "xxx",
  "task_summary": "...",
  "agent_name": "Implementer",
  "workspace_id": "yyy"
}
```

**agent.execution_finished**
```json
{
  "task_id": "xxx",
  "task_summary": "...",
  "agent_name": "Implementer",
  "workspace_id": "yyy"
}
```

#### Chat Events

**chat.message_added**
```json
{
  "chat_id": "xxx",
  "chat_title": "...",
  "author_type": "user|agent|system",
  "workspace_id": "yyy"
}
```

**chat.processing_started**
```json
{
  "chat_id": "xxx",
  "chat_title": "...",
  "agent_name": "Malamar",
  "workspace_id": "yyy"
}
```

**chat.processing_finished**
```json
{
  "chat_id": "xxx",
  "chat_title": "...",
  "agent_name": "Malamar",
  "workspace_id": "yyy"
}
```

### Event Payload Design

Payloads are self-contained for toast display - no refetch needed for the notification itself. Each payload includes:
- Entity ID for linking/navigation
- Human-readable summary for display
- Workspace ID for filtering

---

## Mailgun Integration

Email notifications are sent via fire-and-forget async calls. No separate background job - triggered inline but non-blocking.

### Configuration

| Field | Description |
|-------|-------------|
| API Key | Mailgun API key |
| Domain | Mailgun domain |
| From Email | Sender email address |
| To Email | Recipient email address |

### Test Email

`POST /api/settings/test-email` - Sends a test email to verify configuration.

### Failure Handling

Failures are silently logged to application error log. No user-facing alert for notification failures.