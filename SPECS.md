# Malamar Project

> **Malamar lets you combine the strengths of different AI CLIs into autonomous multi-agent workflows, limited only by your creativity.**

## Problem Statement

### The Manual Workflow Pain

Working with multiple AI CLI tools today is a tedious, manual process. A typical implementation workflow looks like:

1. Spin up Claude Code, instruct it to discover ideas and write a plan file
2. Quit Claude Code, spin up Gemini CLI to implement the plan (more context, cheaper)
3. Quit Gemini CLI, spin up Codex CLI to review Git changes and write a feedback file
4. Quit Codex CLI, spin up Claude Code to verify feedback and update the plan
5. Loop until the result is good enough

This "quit, spin up, instruct, repeat" cycle is:
- **Time-consuming**: Constant context switching between tools
- **Manual**: Cannot be automated; human is the glue between each step
- **Lossy**: Context is lost between sessions

### Why Multiple AI Tools

Each AI CLI has distinct strengths and tradeoffs:

| Tool | Strengths | Tradeoffs |
|------|-----------|-----------|
| **Claude Code** | Powerful, excellent instruction following | Expensive, low usage limits |
| **Gemini CLI** | Cheaper, longer context, better writing/multilingual | Weaker instruction following |
| **Codex CLI** | Good for long working sessions | Can be naive |
| **OpenCode** | Local LLM via Ollama for compliance/PII | Depends on local hardware |

No single tool is best for everything. The optimal workflow combines their strengths while mitigating their weaknesses.

### The Core Value

Malamar's multi-agent approach (Planner → Implementer → Reviewer → Approver) provides value even with a single CLI:

1. **Automatic mistake catching**: Each agent verifies the work of agents before it
2. **Separation of concerns**: Focused, specialized steps instead of one monolithic session
3. **Reduced hallucination**: Shorter, fresh sessions prevent long-context hallucination where AI makes mistakes late in a session that it wouldn't make in a fresh context

### The Vision

With Malamar, the workflow becomes:

1. Set up a workspace with agents configured to your best practices
2. Create a task with detailed requirements
3. Walk away
4. Come back later - everything is done, automatically

Human intervention happens through the "In Review" status, email notifications, and the ability to steer agents via comments - even remotely from a phone via Tailscale.

### Target Users

Malamar is built for developers and power users who:
- Use multiple AI CLI tools and want to orchestrate them
- Want autonomous task completion without constant supervision
- Need flexibility to define their own workflows via agent instructions

## Application Architecture

Malamar is a self-contained, Pocketbase-like application:

1. **Distribution**: Single executable binary, or run via `bunx`/`npx`. Can be installed with Homebrew.
2. **Runtime**: Runs as a background service with a single command (e.g., `./malamar`).
3. **Data Storage**: Uses `~/.malamar` directory to store the SQLite database, files, and configuration.
4. **Web Interface**: Exposes a web UI on port `3456` for managing workspaces, agents, tasks, and settings.
5. **Remote Access**: Can be accessed remotely via Tailscale for mobile access when away from keyboard.
6. **Dependencies**: Zero external dependencies. Everything is self-contained (no Redis, no external database).
7. **Authentication**: No authentication for now. The application is meant to run locally or behind Tailscale.

### Startup and Recovery

When Malamar starts, all items in the task event queue are automatically picked up and resumed. No manual intervention is needed for tasks that were "In Progress" from a previous run.

## Entities

### Workspace

Workspace is the roof of everything in Malamar.
1. Workspace has the title (for me) and the description (for the agents) to understand the overall context of the tasks inside the workspace.
2. Workspace has multiple agents, usually: Planner; Implementer; Reviewer; Approver.
    2.1. Planner is the AI agent to ensure the requirement of the task is already clear enough, by asking me (as comments) or trying to do that by itself by using all the tools provided.
    2.2. Implementer is the AI agent to acutally working on the task, maybe a writing document task, or maybe a coding task, endless of possibility.
    2.3. Reviewer is the AI agent to double check the work of the implementer.
    2.4. Approver checking if the work is good enough, anything else to clarify before requesting for my attention.
3. Workspace has multiple tasks, each task has the summary (title) and the description. I will be the one who create the task, then the AI agents will working with each other fully autonomous to complete the task.
4. Workspace has a working directory setting that determines where agents execute their work.

#### Working Directory

The working directory is configured per workspace. Two scenarios:

**Scenario 1: Static Working Directory**
- User has a pre-configured directory (e.g., a cloned repository with Git setup).
- All agents work on that specific directory.
- Everything happens there: implement, review, approve, and any other operations.
- User selects the directory path in workspace settings.

**Scenario 2: Temp Folder (Default)**
- A newly-created random working directory per task (`/tmp/malamar_tasks_{task_id}`).
- Ensures a clean workspace for each task.
- Good for multi-repo tasks or tasks that require isolation.
- Agents can clone repositories or create files as needed.
- This is the default when creating a new workspace.

For temp folder scenarios, agents learn which repositories or resources to work with from the workspace-level instruction or the task description.

### Agents

Each agent has the instruction for example:
1. Planner: You're researcher, your responsibility to try your best to enriching the task's summary and requirement as much as possible, use all the tools you're provided, to make a plan to implement the task. If you found that the requirement is dangerously not clear enough, comment your questions and move the ticket to "in review" to draw my attention. Your plan should be in detailed and can be verified, comment your plan into the task so the implementer and the reviewer can working and verifying them.
2. Implementer: You're implementer, your responsibility to implement the task based on its description and the plan of the planner, if there is no plan, please do nothing. You're also responsible to verify the feedback of the reviewer, push back if needed to get the agreement on the improvements/fixes via the comments, then implement them if any.
3. Reviewer: You're reviewer, your responsibility to review the task based on its description, the plan of the planner and the working result of the implementer. You will try your best to ensure the working result meet the industrial quality, comment into the task and discuess with the implementer until the works are ready to be shipped.
4. Approver: You're approver, your responsibility is wait until everyone has on the same page and agree that the task is done, you will try your best to verify the result against the task, the plan and the feedback loop, then comment into the task to seeking for clarification if needed. When you see the result is good enough to be shipped, move the ticket to "in review" so the human can be looped into the task to review the result.

Agent also be setup to be assigned into an AI CLI (if many is available), for example:
1. Planner is assigned to Claude Code
2. Implementer is assigned to Codex CLI
3. Reviewr is assigned to Gemini CLI
4. Approver is assigned to Claude Code

The agent is all about the system instruction, where it will pasted into the Claude Code/Gemini CLI/Codex CLI/etc. when running.

#### Agent Identification

Each agent has its own nanoid. The default example agents (Planner, Implementer, Reviewer, Approver) are auto-created per workspace, but users can edit, delete, add, and reorder them.

#### Agent Order

Each agent has an `order` field that must be unique within the workspace. When reordering agents, the client/UI must send all the new valid ordering. The runner executes agents sequentially based on this order.

#### Agent Response Format

When an agent finishes working on a task, it responds with a JSON structure containing one or more actions:

```json
{
  "actions": [
    { "type": "skip" },
    { "type": "comment", "content": "<markdown string>" },
    { "type": "change_status", "status": "in_review" }
  ]
}
```

**Action Types:**
- `skip`: Agent has nothing to do. Use this when the task doesn't require the agent's expertise, another agent should handle it, the task is already complete, or the agent has nothing meaningful to contribute.
- `comment`: Agent did meaningful work and has progress to report. The `content` field contains a markdown summary of what changed.
- `change_status`: Agent requests to move the task to "In Review". This is the only status change allowed from agents.

**Valid Combinations:**
1. `skip` alone - agent has nothing to do
2. `comment` alone - agent did work, loop continues to next agent
3. `comment` + `change_status: "in_review"` - agent comments and requests human attention
4. `change_status: "in_review"` alone - technically valid but discouraged; agents should always explain why human attention is needed

**Important:** An agent should NEVER comment "I have nothing to do" or similar. If there's nothing to do, use `skip`.

### Tasks

Each task has the status, of:
1. Todo: Default, when I just created the task.
2. In progress: The agents are picked up and actively working on it.
3. In review: When the agents agree with each other that there's nothing more to do, or when human attention is needed, the task moves here waiting for user instruction. Only the human can mark a task as "Done".
4. Done: I move the task here just for historical data. Only the human can move tasks to this status.

Most important, the task has the comments, that is the channel for all task's stakeholder to work with the tasks. The comments can coming from:
1. The user
2. The agents
3. The system, for note if there is an error.

#### Comment Fields

Each comment has the following fields:
- `task_id: string` - The task this comment belongs to
- `workspace_id: string` - The workspace this comment belongs to
- `user_id?: string` - NULL if system or agent comment; if present, it is the mock user ID `000000000000000000000` (21 zeros)
- `agent_id?: string` - NULL if system or user comment; contains the agent's nanoid if from an agent
- `content: string` - The comment content in markdown format (free text)
- `created_at: Date`
- `updated_at: Date`

**Note:** Currently there is only one user with no auth, so a single mock user ID (`000000000000000000000`) is used everywhere. Multiple users will be supported later.

**Deleted Agent Handling:** If a comment was made by an agent that has since been deleted, display "(Deleted Agent)" as the author name. The `agent_id` is kept for reference, but the display gracefully handles missing agents.

### Task Router/Runner

Task router takes the responsibility to bind the task with the agent.

#### Task Events

A task event is emitted when:
1. The user creates a new task
2. The user/agent/system comments on a task
3. The task status/properties has been changed (title/description/etc.)

#### Task Event Queue

Each task has an event queue to manage processing.

**Queue Item States:**
- `queued`: Waiting to be picked up
- `in_progress`: Currently being processed
- `completed`: Finished successfully
- `failed`: Processing failed

**Queue Item Fields:**
- `is_priority: boolean` - When true, this item is picked up first (see Priority Override)
- `updated_at: Date` - Used for LIFO ordering

**Queue Behavior:**
1. Whenever a new task event is emitted, if the queue for that task is empty, a new item is added with status "queued"
2. When the runner picks up a queue item, its status changes to "in_progress"
3. If a new task event is emitted while a queue item is "in_progress", a new item is added with status "queued"
4. If a new task event is emitted while there's already both an "in_progress" and a "queued" item for that task, no new queue item is added - but the existing "queued" item's `updated_at` is refreshed to bump its priority
5. Completed and failed items are kept (for pickup priority logic) and cleaned up by the Queue Cleanup job after 7 days

#### Queue Pickup Priority

The runner picks up queue items in the following order (per workspace):

1. **Priority flag first**: Any queue item with `is_priority: true` and status `queued`
2. **Most recently processed task**: Find the task that most recently has a `completed` or `failed` queue item, then pick its `queued` item
3. **LIFO fallback**: If no recent history, pick the most recently updated `queued` item (by `updated_at`)

**Rationale**: Tackle one task until completion, rather than starting many tasks but finishing none. LIFO ordering allows users to "bump" a task by commenting on it (which updates the queue item's `updated_at`).

#### Priority Override

Users can manually prioritize a task for the next loop via a button in the UI:

1. Find or create a queue item for that task
2. Set `is_priority: true` on that queue item
3. Set `is_priority: false` on all other queue items in that workspace
4. The runner will pick up this task next

Only one task can be prioritized at a time per workspace. After processing, the runner returns to normal pickup behavior.

#### Status Transitions

**Runner-triggered transitions** (when a task event is picked up):
1. If the task is in "Todo", move it to "In Progress"
2. If the task is in "In Progress" but all agents skip (no comments added), move it to "In Review" automatically
3. If the task is in "In Review" but there is a new comment from the user, move it back to "In Progress" and retrigger the loop
4. If the task is in "Done", do nothing

**User-triggered transitions:**
- User can move a task from "Done" back to "Todo" or "In Progress" (after commenting or editing the task)
- User can move a task to "Done" (only humans can mark tasks as done)
- No restrictions on user-initiated status changes

**Auto-demotion:**
- When the runner picks a task for a new loop, all OTHER "In Progress" tasks in that workspace are automatically moved to "Todo"
- This ensures only one task shows as "In Progress" at a time per workspace

#### The Loop

This is the core concept of Malamar:

1. The runner continuously picks up queue items, but each workspace only has one queue item being processed at a time.
2. After picking up a queue item, the runner collects all task data and workspace data, then routes to each agent sequentially (based on agent order).
3. For each agent, Malamar triggers the assigned CLI (e.g., Claude Code), passing in the task, task comments, and workspace instruction. The working directory is either a newly created temp folder (`/tmp/malamar_tasks_{task_id}`) or a static working directory configured in the workspace settings.
4. The agent works on the task and responds with actions (skip, comment, change_status). The runner executes those actions, then moves to the next agent. Since agents work in the same directory, they can indirectly share working resources with each other.
5. After all agents finish the current pass:
   - If any agent added a comment, the runner retriggers the loop from the first agent with the updated task state
   - If all agents chose "skip" and no new comments were added, the loop ends and the task moves to "In Review"
6. When retriggered, the loop always starts from the first agent in the workspace order.

#### Agent Execution Order

The runner uses a **just-in-time snapshot** approach for agent execution:

1. Before executing each agent, the runner queries for the next agent by finding the agent with the smallest `order` value greater than the current agent's `order`.
2. Agent data (instruction, name, assigned CLI) is fetched right before execution, not at the start of the loop.
3. This means changes to agents take effect immediately for subsequent agents in the current loop.

**Implications:**
- If an agent is deleted mid-loop, it's simply not found and skipped.
- If a new agent is inserted with an `order` between the current and next agent, it will be picked up.
- Changes to an agent's instruction don't affect an already-running execution.

#### Concurrent Processing

The runner uses a single process with concurrent workers per workspace:

1. Each workspace has its own worker that handles queue pickup and agent loops.
2. Multiple workspaces can process tasks simultaneously.
3. Within a workspace, only one task is processed at a time.
4. There is no artificial limit on concurrent workspace workers.

#### Agents on "In Review" Tasks

Agents are NOT allowed to run on tasks that are already in "In Review". If a queue item is picked up but the task is in "In Review", no agents will execute.

#### Agent Comment and Status Change

If an agent both comments AND requests "In Review" in the same response:
- The runner stops the current loop immediately
- The comment is saved and the task moves to "In Review"
- No subsequent agents in the current pass will run

#### Error Handling

The following errors are handled uniformly:
- **CLI execution failure**: Process crashes or returns non-zero exit code
- **CLI timeout**: Process takes too long (handled by OS)
- **Malformed output**: Output file is empty, missing, contains invalid JSON, or doesn't match the expected schema

When any of these errors occur:
1. The runner writes a System comment with detailed error information (e.g., "Output file was empty", "Invalid JSON: unexpected token at position 42", "CLI exited with code 1")
2. The current loop stops immediately
3. The System comment emits a task event, which adds a new queue item
4. The task does NOT move to "In Review" - a new loop will be triggered through the queue (contextual retry)
5. There is no explicit retry backoff; the task event queue provides natural delay

This allows agents to see the error in subsequent loops and potentially self-correct.

#### Iteration Limits

There is no hard cap on iteration count or time. The loop continues until:
- All agents skip (nothing to do), OR
- An agent requests "In Review", OR
- An error occurs (which triggers a retry via the queue)

## CLI Adapters

Malamar supports multiple AI CLI tools through an adapter pattern. Each supported CLI has its own adapter with hardcoded configuration for command templates, flags, and capabilities.

### Supported CLIs

| CLI | Binary Name |
|-----|-------------|
| Claude Code | `claude` |
| Gemini CLI | `gemini` |
| OpenAI Codex CLI | `codex` |
| OpenCode | `opencode` |

### Auto-Detection

On startup and periodically (every 5 minutes), Malamar automatically detects available CLIs:

1. Search for the binary in PATH (or use custom path from settings)
2. Run a simple test prompt (e.g., "Respond with OK")
3. Verify the process exits successfully with non-empty output
4. Store the detected version in memory

This ensures the CLI is both installed AND properly configured (e.g., API keys set).

### CLI Settings

Each CLI has a settings section in the CLI Settings page:

| Field | Description |
|-------|-------------|
| Binary Path | Custom path override (empty = use PATH) |
| Environment Variables | Key-value pairs to inject when running the CLI |
| Detected Version | Read-only, shows auto-detected version |
| Status | Read-only: "Available", "Not Found", or "Test Failed" |

### CLI Unavailability Warnings

When an assigned CLI becomes unavailable, warnings are displayed on:
- Workspace detail page
- Create task form
- Task detail page

This gives users visibility before they create or run tasks.

### CLI Invocation

Each CLI is invoked via direct shell command with the configured environment variables. For example:

```shell
claude --dangerously-skip-permissions --json-schema ... --output-format json --prompt ...
```

The runner waits for the subprocess to exit before proceeding.

### Response Format Enforcement

- CLIs that support JSON schema flags (e.g., Claude Code) use native schema enforcement for the agent response format.
- CLIs without schema support have the response format instruction embedded in the prompt.

### CLI Unavailability at Runtime

Agent assignment to a CLI is allowed even if the CLI isn't currently available. At runtime, if an assigned CLI isn't available, the runner treats it as an error:
1. System comment is added to the task with error details.
2. Standard error handling flow applies (new queue item, retry via queue).

## Context Passing

When the runner invokes an agent's CLI, it passes context through temporary files.

### Input File

The runner creates an input file at `/tmp/malamar_task_{task_id}.md` (overwritten for each agent execution). This file contains:

1. **Malamar Context**: Information that the agent is being controlled by Malamar, plus the workspace instruction.
2. **Agent Instruction**: The agent's own instruction, plus a brief of other agents in the same workspace (names only, no instructions).
3. **Task Details**: The task summary, description, and all comments.
4. **Output Instruction**: Instruction to write the result to a pre-created output file.

The CLI is invoked with an instruction like: "Read the file at $FILE_PATH and follow the instruction autonomously."

#### Input File Format

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

# Output Instruction
Write your response as JSON to: /tmp/malamar_output_{random_nanoid}.json
```

**Comment Format Notes:**
- Comments are in JSONL format (one JSON object per line) inside a code block.
- This prevents markdown content in comments from breaking the file structure.
- The `author` field values:
  - Agent comments: the agent's name at the time the comment was created
  - System comments: "System"
  - User comments: "User"
- The "Other Agents" list reflects agents at the moment of input file creation.

### Output File

The agent writes its response (the actions JSON) to a pre-created file at `/tmp/malamar_output_{random_nanoid}.json`. The runner reads this file after the subprocess exits.

## Web UI

The web interface is exposed on port `3456` and provides access to all Malamar features.

### Task Creation

Task creation is a simple form with:
- **Summary**: Short text (title)
- **Description**: Markdown content

### Real-Time Updates

Two mechanisms for keeping the UI in sync:

1. **Task-Level Polling**: The web UI polls every 3 seconds (using React Query or similar) to gather new task data. This approach works well when the user is actively viewing or commenting on a task.

2. **Global SSE Endpoint**: The web UI connects to a Server-Sent Events endpoint for real-time notifications/toasts when events happen in the backend. The backend uses an in-process pub/sub event emitter to fan out events.

#### SSE Events

| Event | Payload |
|-------|---------|
| `task.status_changed` | task_id, task_summary, old_status, new_status, workspace_id |
| `task.comment_added` | task_id, task_summary, author_name, workspace_id |
| `task.error_occurred` | task_id, task_summary, error_message, workspace_id |
| `agent.execution_started` | task_id, task_summary, agent_name, workspace_id |
| `agent.execution_finished` | task_id, task_summary, agent_name, workspace_id |

Payloads are self-contained for toast display - no refetch needed for the notification itself.

## Notifications

Malamar can send email notifications via Mailgun API when certain events occur.

### Mailgun Configuration

Global settings in the Settings page:

| Field | Description |
|-------|-------------|
| API Key | Mailgun API key |
| Domain | Mailgun domain |
| From Email | Sender email address |
| To Email | Recipient email address |

A "Send Test Email" button allows users to verify the configuration works.

### Configurable Events

Users can enable/disable notifications per event type in the settings page:

1. **On error occurred**: Enabled by default
2. **On task moved to in review**: Enabled by default

### Settings Scope

Notification settings are global with per-workspace overrides:
- **Global**: Mailgun configuration (API key, domain, from/to email) applies to all workspaces
- **Per-workspace**: Toggle events on/off (inherits global email config)

Workspace settings only control which events trigger notifications, not the email configuration itself.

## User Actions

### Kill Loop

Users can manually kill the current loop of a task if it's stuck or taking too long. When a loop is killed:

1. Send SIGTERM to the running CLI subprocess.
2. Leave the output file as-is (no cleanup).
3. A system comment is added to the task noting that the user canceled the loop.
4. The task moves to "In Review" so the user can investigate the problem.
5. The user can move the task back to "In Progress" when ready to retry.

### Delete Workspace

Users can delete a workspace entirely. This is a destructive operation:

1. User must type the workspace name to confirm.
2. If a loop is running, send SIGTERM to the CLI subprocess.
3. Cascade delete all related data: agents, tasks, comments, queue items.

### Delete Task

Users can delete a task. This follows the same pattern as workspace deletion:

1. User must confirm the deletion.
2. If the task has an active loop, send SIGTERM to the CLI subprocess.
3. Delete the task and all its comments and queue items.

## Background Jobs

Malamar runs several background jobs:

### Runner

The main job that processes tasks:
- Runs as a continuous loop with 1 second sleep between checks.
- Spawns concurrent workers per workspace (see Concurrent Processing).
- Picks up queue items based on priority order.

### Queue Cleanup

Cleans up old queue items to prevent table overflow:
- Runs on a cron schedule (e.g., daily).
- Deletes `completed` and `failed` queue items older than 7 days.
- Leaves `queued` and `in_progress` items untouched.

### CLI Health Check

Periodically verifies CLI availability:
- Runs every 5 minutes.
- Updates in-memory CLI status.
- UI polls a health endpoint to get current status (no SSE push for CLI changes).

### Email Notifications

Email notifications are sent via fire-and-forget async calls when events occur. This is not a separate background job - notifications are triggered inline but don't block the runner.

## Technical Decisions

### Domain Agnostic Design

Malamar is purely an orchestration layer for multi-agent CLI workflows. It has no opinions on what agents do - all domain-specific behavior is defined in agent instructions. Use cases include but are not limited to:

- Software development (coding, reviewing, pushing)
- Writing blog posts
- Browser automation via CLI
- System management
- HR spreadsheet work
- Any task achievable via AI CLIs

Malamar only cares about the agent response format so the runner can understand the outcome.

### Miscellaneous

1. All IDs should be in nanoid format.
2. The mock user ID is `000000000000000000000` (21 zeros, a valid nanoid).
3. No explicit timeout for CLI execution - let the OS handle it. Users can manually kill loops if needed.

## Aim Far

Future enhancements to consider (not for initial implementation):

### S3 Backup

S3-compatible automatic backup for the SQLite database and files in `~/.malamar`.

### Import/Export

Ability to export workspaces, tasks, and settings for backup or migration, and import them into another Malamar instance.

### Related Tasks Query

Allow agents to query Malamar's API (via CURL to the running port) to collect additional information such as related tasks in the same workspace. This would be passed as additional context in the input file.
