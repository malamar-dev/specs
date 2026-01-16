# Malamar Project

Malamar is a personal project of Irving Dinh (me), it is designed to solve my problems:
1. I've Claude Code, Gemini CLI and also Codex CLI.
2. I want them working with each other, to checking other works, back to back. When everything is done, it will move to the in review and wait for my touching.

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

### Task Router/Runner

Task router takes the responsibility to bind the task with the agent.

#### Task Events

A task event is emitted when:
1. The user creates a new task
2. The user/agent/system comments on a task
3. The task status/properties has been changed (title/description/etc.)

#### Task Event Queue

Each task has an event queue to manage processing:
1. Whenever a new task event is emitted, if the queue for that task is empty, a new item is added with status "queued"
2. When the runner picks up a queue item, its status changes to "in_progress"
3. If a new task event is emitted while a queue item is "in_progress", a new item is added with status "queued"
4. If a new task event is emitted while there's already both an "in_progress" and a "queued" item for that task, no new queue item is added

#### Status Transitions

When a task event is picked up:
1. If the task is in "Todo", move it to "In Progress"
2. If the task is in "In Progress" but all agents skip (no comments added), move it to "In Review" automatically
3. If the task is in "In Review" but there is a new comment from the user, move it back to "In Progress" and retrigger the loop
4. If the task is in "Done", do nothing

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

#### Agents on "In Review" Tasks

Agents are NOT allowed to run on tasks that are already in "In Review". If a queue item is picked up but the task is in "In Review", no agents will execute.

#### Agent Comment and Status Change

If an agent both comments AND requests "In Review" in the same response:
- The runner stops the current loop immediately
- The comment is saved and the task moves to "In Review"
- No subsequent agents in the current pass will run

#### Error Handling

If an agent fails or times out:
1. The runner writes a System comment with detailed error/timeout information (free text format)
2. A new task event is emitted into the queue
3. The current loop stops
4. The task does NOT move to "In Review" - a new loop will be triggered through the queue (contextual retry)
5. There is no explicit retry backoff; the task event queue provides natural delay

#### Iteration Limits

There is no hard cap on iteration count or time. The loop continues until:
- All agents skip (nothing to do), OR
- An agent requests "In Review", OR
- An error occurs (which triggers a retry via the queue)

## CLI Adapters

Malamar supports multiple AI CLI tools through an adapter pattern. Each supported CLI has its own adapter with hardcoded configuration for command templates, flags, and capabilities.

### Supported CLIs

1. Claude Code
2. Gemini CLI
3. OpenAI Codex CLI
4. OpenCode

### Auto-Detection

On startup, Malamar automatically searches for available CLIs from the supported list and shows them to the user. Only detected CLIs can be assigned to agents.

### CLI Invocation

Each CLI is invoked via direct shell command. For example:

```shell
claude --dangerously-skip-permissions --json-schema ... --output-format json --prompt ...
```

The runner waits for the subprocess to exit before proceeding.

### Response Format Enforcement

- CLIs that support JSON schema flags (e.g., Claude Code) use native schema enforcement for the agent response format.
- CLIs without schema support have the response format instruction embedded in the prompt.

### CLI Unavailability

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

2. **Global SSE Endpoint**: The web UI connects to a Server-Sent Events endpoint for real-time notifications/toasts when events happen in the backend (e.g., new comment, status change, error occurred). The backend uses an in-process pub/sub event emitter to fan out events.

## Notifications

Malamar can send email notifications via Mailgun API when certain events occur.

### Configurable Events

Users can enable/disable notifications per event type in the settings page:

1. **On error occurred**: Enabled by default
2. **On task moved to in review**: Enabled by default

### Settings Scope

Notification settings are global with per-workspace overrides. Global settings apply to all workspaces unless a workspace explicitly overrides them.

## User Actions

### Kill Loop

Users can manually kill the current loop of a task if it's stuck or taking too long. When a loop is killed:

1. A system comment is added to the task noting that the user canceled the loop.
2. The task moves to "In Review" so the user can investigate the problem.
3. The user can move the task back to "In Progress" when ready to retry.

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
