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

**Working Directory Conflicts:** Malamar does not prevent multiple workspaces from using the same static directory path. Users are responsible for avoiding conflicts if they configure multiple workspaces to point at the same directory.

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

#### Agent Instructions Responsibility

Writing effective agent instructions is the **user's responsibility**. The user must define:
- **WHEN** to skip, comment, or move a task to "In Review"
- **WHAT** the agent should focus on and verify

Malamar's responsibility is the **HOW**:
- Ensuring agents respond in the correct format
- Executing the actions agents return
- Managing the loop and queue mechanics

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

**Important:** When creating a task, users must include all context in the task's DESCRIPTION field. Comments are NOT for adding initial context - they are only used to steer agents during processing. The runner may pick up a task immediately after creation, so all necessary information must be in the description from the start.

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

### Activity Logs

Activity logs provide a linear audit trail of everything that happens to a task. They are stored in a separate `task_logs` table alongside `task_comments`.

#### Activity Log Fields

Each activity log entry has the following fields:
- `task_id: string` - The task this log belongs to
- `workspace_id: string` - The workspace this log belongs to
- `event_type: string` - The type of event (e.g., "created", "status_changed", "comment_added", "agent_started", "agent_finished")
- `actor_type: string` - Who triggered the event ("user", "agent", or "system")
- `actor_id?: string` - The user_id or agent_id (nullable for system events)
- `metadata: JSON` - Event-specific data (e.g., `{"old_status": "todo", "new_status": "in_progress"}`)
- `created_at: Date`

#### Tracked Events

Activity logs are created for:
- Task created
- Task status changed (with old/new status in metadata)
- Comment added
- Agent execution started
- Agent execution finished
- Task properties edited (title, description)
- Loop canceled by user

### Chat

The Chat feature allows users to have conversations with agents within a workspace. Users can chat with:
- **Any workspace agent** - For ad-hoc assistance related to that agent's expertise
- **The Malamar agent** - A special built-in agent designed to help manage the workspace, create/tune agents, and answer questions about Malamar

#### Chat Entity

Each chat belongs to a workspace and is associated with either a workspace agent or the Malamar agent.

**Chat Fields:**
- `id: string` - Unique identifier (nanoid)
- `workspace_id: string` - The workspace this chat belongs to
- `agent_id?: string` - The workspace agent (NULL if chatting with Malamar agent)
- `cli_type?: string` - CLI override (NULL = respect agent's CLI or Malamar default)
- `title: string` - Chat title (default: "Untitled chat")
- `created_at: datetime`
- `updated_at: datetime`

**CLI Selection Behavior:**
1. **Malamar agent** (agent_id is NULL): Uses first available CLI in order: Claude Code → Gemini CLI → Codex CLI → OpenCode
2. **Workspace agents**: Uses the CLI configured on the agent
3. **CLI override** (cli_type is not NULL): Overrides both of the above

#### Chat Messages

**Chat Message Fields:**
- `id: string` - Unique identifier (nanoid)
- `chat_id: string` - The chat this message belongs to
- `role: enum` - "user", "agent", or "system"
- `message: string` - The conversational text
- `actions: json?` - NULL for user/system messages; optional array for agent messages
- `created_at: datetime`

**System messages** are used for:
- Error information (e.g., "Error: CLI exited with code 1")
- File attachment notifications (e.g., "User has uploaded /tmp/malamar_chat_{chat_id}_attachments/{filename}")

#### Chat Actions

When an agent responds in a chat, it can include actions alongside its message:

```json
{
  "message": "I've created a Reviewer agent for TypeScript code review.",
  "actions": [
    { "type": "create_agent", "name": "Reviewer", "instruction": "...", "cli_type": "claude", "order": 3 }
  ]
}
```

The `message` field is encouraged but not required. Actions are displayed as structured cards/badges below the message text in the UI.

**Action Types - All Chat Agents:**
- `rename_chat` - Set the chat title: `{ "type": "rename_chat", "title": "..." }`
- `create_task` - Create a new task: `{ "type": "create_task", "summary": "...", "description": "..." }`

**Action Types - Malamar Agent Only:**
- `create_agent` - Create a new agent: `{ "type": "create_agent", "name": "...", "instruction": "...", "cli_type": "claude|gemini|codex|opencode", "order": number }`
- `update_agent` - Modify an agent: `{ "type": "update_agent", "agent_id": "...", "name?": "...", "instruction?": "...", "cli_type?": "..." }`
- `delete_agent` - Remove an agent: `{ "type": "delete_agent", "agent_id": "..." }`
- `reorder_agents` - Change execution order: `{ "type": "reorder_agents", "agent_ids": ["id1", "id2", "id3"] }`
- `update_workspace` - Modify workspace settings: `{ "type": "update_workspace", "title?": "...", "description?": "...", ... }`

**Action Execution:** Actions are executed immediately when the agent responds - no confirmation step. If an action fails (e.g., duplicate agent name, invalid agent ID), a system message is added with the error details.

#### Chat Processing

Chat processing is similar to task processing:

1. User sends a message
2. Malamar collects all messages in the chat plus metadata (agent instruction, workspace context)
3. Serialize into a markdown file at `/tmp/malamar_chat_{chat_id}.md`
4. Instruct the CLI to read from it and respond in a format Malamar can parse
5. CLI writes response to `/tmp/malamar_chat_{chat_id}_output.json`
6. Malamar parses the response, executes any actions, and displays the message

**Working Directory:** The chat respects the workspace's working directory setting:
- **Temp Folder mode**: Creates `/tmp/malamar_chat_{chat_id}` as the working directory
- **Static Directory mode**: Uses the directory defined in the workspace settings

This enables agents to work with full context of the workspace. For example, if the workspace points to a repository, the agent can browse files, run commands, and make changes during the chat.

**Chat Queue:** Chat processing is tracked via a `chat_queue` table with the same states as task queue (queued, in_progress, completed, failed). No PID storage needed - the processor holds the subprocess reference in memory.

**Concurrency:**
- Chats within one workspace CAN process concurrently (unlike tasks which are one-at-a-time)
- No limit on concurrent chat processing across workspaces
- UI prevents sending multiple messages in the same chat while a response is pending

**Error Handling:** Errors are displayed as system messages in the chat (e.g., "Error: CLI exited with code 1" or "Error: Output file was empty"). System messages are visible to users and included in the input file when the conversation continues.

#### Malamar Agent

The Malamar agent is a special built-in agent that helps users manage their workspace:
- Create, update, delete, and reorder agents
- Update workspace settings
- Answer questions about how to use Malamar
- Help users write effective agent instructions

**Instruction Source:** The Malamar agent's instruction is hardcoded in Malamar's codebase, with extra instructions pointing to a remote knowledge base URL (e.g., `https://github.com/malamar-dev/specs/blob/main/AGENTS/README.md`) for discovering best practices and documentation.

**Context File:** When chatting with any agent (including Malamar), a context file at `/tmp/malamar_chat_{chat_id}_context.md` contains:
- Workspace settings (title, description, working directory, cleanup settings, notifications)
- All agents with their IDs and instructions (ordered), e.g., "Planner (id: 123456abcdef)"
- Global settings (available CLIs and their health status, Mailgun configuration status)

The agent is instructed to read this file ONLY when needed. Task summaries are NOT included.

#### File Attachments

Users can upload files into a chat:

1. User uploads a file (any format, no size limit)
2. File is placed at `/tmp/malamar_chat_{chat_id}_attachments/{filename}`
3. A system message is created: "User has uploaded /tmp/malamar_chat_{chat_id}_attachments/{filename}"
4. The agent can reference the file in subsequent responses

**Multiple files:** Each file creates its own system message.

**Duplicate filenames:** Overwrites the existing file (no delete attachment functionality).

**Cleanup:** Attachment directories are left for the OS to clean up naturally via `/tmp` cleanup, consistent with other temp files.

#### Chat and Agent Deletion

**When an agent is deleted:** Any chat using that agent is automatically switched to the Malamar agent. Users can also manually change the chat's agent via a dropdown in the chat UI.

**When a chat is deleted:** Simple confirm popup. The chat and all messages are deleted. Attachment directory is left for OS cleanup.

**When a workspace is deleted:** All chats and chat messages are cascade deleted.

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

1. **Status filter first**: Only consider queue items where the associated task has status "Todo" or "In Progress". Queue items for tasks in "In Review" or "Done" are ignored until the status changes back. This prevents wasted runner cycles.
2. **Priority flag**: Any queue item with `is_priority: true` and status `queued`
3. **Most recently processed task**: Find the task that most recently has a `completed` or `failed` queue item, then pick its `queued` item
4. **LIFO fallback**: If no recent history, pick the most recently updated `queued` item (by `updated_at`)

**Rationale**: Tackle one task until completion, rather than starting many tasks but finishing none. LIFO ordering allows users to "bump" a task by commenting on it (which updates the queue item's `updated_at`).

#### Priority Override

Users can manually prioritize a task for the next loop via a button in the UI:

1. Find or create a queue item for that task
2. Set `is_priority: true` on that queue item
3. Set `is_priority: false` on all other queue items in that workspace
4. The runner will pick up this task next

Only one task can be prioritized at a time per workspace. After processing, the runner returns to normal pickup behavior.

**Behavior with running tasks:** If a task is prioritized while another task is mid-loop in "In Progress", the prioritized task waits until the current loop finishes. Prioritization does NOT interrupt or cancel running tasks. If urgent, the user should prioritize the needed task AND manually cancel the currently running task.

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

**Queue Item Creation:** Moving a task to "In Progress" or "Todo" (whether by user or system) always creates a queue item if one doesn't already exist. This ensures the runner always picks up tasks that need processing.

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
| Status | Read-only: "Healthy" or "Unhealthy" |
| Error Message | Read-only, optional: details when unhealthy (e.g., "binary not found in PATH", "test prompt returned empty response") |

**Status Simplification:** CLI status is simplified to two states: "Healthy" (CLI is available and working) and "Unhealthy" (CLI is not found or test failed). The optional error message field provides more detail for troubleshooting.

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

When the runner invokes an agent's CLI, it passes context through temporary files:

- **Input file**: A markdown file containing workspace context, agent instruction, other agents in the workflow, task details (summary, description, comments), activity logs, and output instructions.
- **Output file**: A pre-created JSON file where the agent writes its response (the actions JSON).

The CLI is invoked with an instruction to read the input file and follow the instructions autonomously. The runner reads the output file after the subprocess exits.

For the detailed file format specification and examples, see [TECHNICAL_DESIGN.md#context-passing](./TECHNICAL_DESIGN.md#context-passing).

## Web UI

The web interface is exposed on port `3456` and provides access to all Malamar features.

### Workspace List Page

The home page displays a list of workspaces as cards. Each workspace card shows:
- Workspace name
- Number of agents
- Number of tasks in Todo / In Progress / In Review (Done count is not shown on the list page)

**Sorting Options:**
1. **Recently active** (default): Sorted by `last_activity_at` descending. This column is updated whenever any tracked event occurs in the workspace (task created, status changed, comment added, agent execution started/finished, etc.).
2. **Created date**: ASC or DESC by `created_at`
3. **Updated date**: ASC or DESC by `updated_at`

### Workspace Detail Page

When accessing a workspace, the user sees tabs for different views:

#### Tasks Tab (Kanban Board)

The default view is a Kanban board of tasks, similar to JIRA, Trello, or Linear.

**Columns:**
- Four hardcoded columns mapping directly to task statuses: Todo, In Progress, In Review, Done
- No grouping or customization for now

**Task Ordering:**
- Within each column, tasks are ordered by most recently updated on top

**Interactions:**
- Drag-and-drop between columns is NOT supported - status changes only happen through the task detail popup
- Clicking a task opens the task detail popup

#### Chat Tab

The Chat tab displays a list of previous chats, ordered by recently active (chats with recent messages at the top).

**Chat List Display:**
Each chat item shows:
- Title
- Agent name
- Last message/action preview
- Timestamp of last activity (human diff format, e.g., "2 hours ago")

**Creating New Chats:**
A dropdown button allows creating a new chat. Selectable items are:
- Malamar Agent
- All workspace agents

**Chat Detail Popup:**
Clicking a chat opens a popup with:
- **Header**: Chat title (editable inline by clicking) and agent name (as a dropdown to change agents)
- **CLI Selector**: Dropdown next to agent selector to override which CLI handles the chat
- **Message List**: All messages displayed with oldest on top (natural reading order)
- **Input Area**: Text input with attachment icon and send/stop button

**Agent and CLI Selection:**
- Agent dropdown and CLI dropdown are side by side in the header
- Changing agent → auto-updates CLI dropdown to match that agent's configured CLI (sets `cli_type` to NULL)
- Manually changing CLI → saves `cli_type` override to the chat
- Changing the agent does NOT affect conversation history - only future messages use the new agent

**Processing Indicator:**
While a chat is processing:
- Send button changes to a stop button (with pulse animation)
- User cannot send another message until processing completes
- Clicking stop cancels the processing (kills CLI subprocess, marks queue item as failed)

**File Attachments:**
- Attachment icon next to the message input box
- Clicking opens file picker (any file format, no size limit)
- Uploaded files appear as system messages in the chat

#### Agents Tab

(Existing agent management functionality)

### Task Detail Popup

When selecting a task from the Kanban board, a popup displays the task's detail. The layout is similar to JIRA or Linear:

1. **Header**: Task title (editable)
2. **Description**: Markdown content (editable)
3. **Tab Section**: Switch between "Comments" and "Activity" tabs (dropdown on mobile)
   - **Comments tab**: Shows all comments in DESC order (newest on top)
   - **Activity tab**: Shows all activity logs in DESC order (newest on top)
4. **Action Buttons**: Available actions based on current status (see Task Action Buttons section)

### Task Creation

Task creation is a simple form with:
- **Summary**: Short text (title)
- **Description**: Markdown content

### Workspace Settings Page

A dedicated settings page for each workspace with the following sections:

**1. General**
- Title: The workspace name (for the user)
- Description/Instruction: Context for agents (the workspace-level instruction)

**2. Working Directory**
- Mode: Static Directory or Temp Folder (default: Temp Folder)
- Directory Path: Shown only when Static Directory mode is selected

**3. Task Cleanup**
- Auto-delete Done tasks: ON/OFF toggle (default: ON)
- Retention period: Number of days before auto-deletion (default: 7, set to 0 to disable)

**4. Notifications**
- On error occurred: ON/OFF (inherits global default)
- On task moved to In Review: ON/OFF (inherits global default)

**Note:** Task cleanup settings are per-workspace with no global default configuration. The 7-day retention is hardcoded as the default for new workspaces.

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
| `task.created` | task_id, task_summary, workspace_id (triggered when created via chat action) |
| `agent.execution_started` | task_id, task_summary, agent_name, workspace_id |
| `agent.execution_finished` | task_id, task_summary, agent_name, workspace_id |
| `chat.message_added` | chat_id, chat_title, author_type (user/agent/system), workspace_id |
| `chat.processing_started` | chat_id, chat_title, agent_name, workspace_id |
| `chat.processing_finished` | chat_id, chat_title, agent_name, workspace_id |

Payloads are self-contained for toast display - no refetch needed for the notification itself.

**Task Created via Chat:** When an agent uses the `create_task` action from chat, a `task.created` SSE event is emitted. The UI shows a toast notification with a clickable link to open the newly created task.

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

### Task Action Buttons

Different actions are available depending on the task's current status:

**Todo:**
- **Delete**: Remove the task entirely
- **Prioritize**: Force this task to the top of the queue for the next pickup

**In Progress:**
- **Cancel**: Kill the current loop (sends SIGTERM to CLI subprocess). The task stays in "In Progress" and will be re-picked by the runner. Use "Move to In Review" afterward if you want to pause processing.
- **Move to In Review**: Manually move the task to await human attention
- **Prioritize**: Force this task to the top of the queue
- No "Delete" action - cancel first, move to In Review, then delete if needed

**In Review:**
- **Move to Todo**: Return the task to the backlog
- **Delete**: Remove the task entirely

**Done:**
- **Move to Todo**: Reopen the task
- **Delete**: Remove the task entirely

### Delete All Done Tasks

Users can bulk-delete all tasks in the "Done" column for a workspace:

1. User clicks "Delete all Done tasks" action (available on the workspace/board view)
2. User must type the workspace name to confirm (similar to workspace deletion)
3. All tasks with "Done" status in that workspace are deleted
4. Cascade deletes all associated data: comments, activity logs, queue items

This is a destructive action requiring confirmation because it affects multiple tasks at once.

### Cancel Loop

Users can manually cancel the current loop of a task if it's stuck or taking too long. When a loop is canceled:

1. Send SIGTERM to the running CLI subprocess.
2. Leave the output file as-is (no cleanup).
3. A system comment is added to the task noting that the user canceled the loop.
4. The task stays in "In Progress" - it does NOT automatically move to "In Review".
5. The runner will re-pick the task following normal queue rules.
6. If the user wants to pause processing, they should manually "Move to In Review" after canceling.

**Child Processes:** Malamar is only responsible for terminating the CLI process itself. If the CLI has spawned child processes, those are the CLI's responsibility to handle gracefully - Malamar cannot control them.

**Use cases for Cancel:**
- Un-hang an agent/CLI that appears stuck
- Make time to rethink the approach before the next loop picks it up

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

Malamar runs several background jobs to keep the system running smoothly:

- **Runner**: The main job that continuously processes tasks, picking up queue items and executing agent loops.
- **Cleanup Job**: A daily job that:
  - Cleans up old task queue items (completed/failed older than 7 days)
  - Cleans up old chat queue items (completed/failed older than 7 days)
  - Auto-deletes Done tasks based on workspace retention settings
- **CLI Health Check**: Periodically verifies CLI availability and updates status.

For implementation details including polling intervals, concurrent processing, and cleanup logic, see [TECHNICAL_DESIGN.md#background-jobs](./TECHNICAL_DESIGN.md#background-jobs).

## Aim Far

Future enhancements to consider (not for initial implementation):

### S3 Backup

S3-compatible automatic backup for the SQLite database and files in `~/.malamar`.

### Import/Export

Ability to export workspaces, tasks, and settings for backup or migration, and import them into another Malamar instance.

### Related Tasks Query

Allow agents to query Malamar's API (via CURL to the running port) to collect additional information such as related tasks in the same workspace. This would be passed as additional context in the input file.

### In-Browser Push Notifications

Add in-browser push notifications as a notification channel. Would use the same configurable events as email notifications (on error occurred, on task moved to in review) to keep configuration simple and consistent.

### Clone Workspace

Ability to clone a workspace, including its settings and agents, but starting fresh with no tasks. Useful for creating templates for similar types of work. May also extend to cloning individual agents or tasks.

### Agent Client Protocol (ACP)

Investigate using the Agent Client Protocol (ACP, https://zed.dev/acp) as an alternative to calling AI CLIs via subprocess commands. This could provide a more standardized interface for agent communication.

### Community Gallery of Agent Configurations

A gallery where users can share and discover agent configurations created by the community. Users could browse, import, and adapt agent setups for different use cases (code review, blog writing, documentation, etc.).
