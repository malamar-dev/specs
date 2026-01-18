# Session 002

Please read [SKILL.md](../.claude/skills/explore-specs/SKILL.md) to understand the process used to create this file.

## Question #1: CLI Invocation Method

How does Malamar invoke the CLI tools (Claude Code, Gemini CLI, Codex CLI)?

### Answer

Malamar uses direct shell invocation. For example:
```shell
claude --dangerously-skip-permissions --json-schema ... --output-format json --prompt ...
```

The CLI is spawned as a subprocess with appropriate flags and arguments configured per CLI type.

## Question #2: JSON Schema Flag Support

For the `--json-schema` flag, is Malamar providing a schema that enforces the agent response format?

### Answer

It depends on the CLI. Claude Code supports the `--json-schema` flag natively, so Malamar can enforce the response format directly. Other CLIs that don't support this flag will have the response format instruction embedded in the prompt itself.

## Question #3: CLI Feature Detection

How does Malamar know which CLI supports what features?

### Answer

Each supported CLI (Claude Code, Gemini CLI, OpenAI Codex CLI, OpenCode) is treated as an adapter with case-by-case hardcoded configuration for:
- Command flags
- Command templates
- Timeout settings
- Feature support (e.g., JSON schema enforcement)

On startup, Malamar auto-detects which CLIs are available on the system.

## Question #4: Unavailable CLI Assignment

What happens if an agent is assigned to a CLI that isn't available?

### Answer

The assignment is allowed at configuration time, but at runtime if the CLI isn't available, the runner treats it as an error. The error is handled through the standard error flow:
1. System comment with error details is added to the task
2. New queue item is created
3. Retry via queue continues

## Question #5: Notification System

Is there a notification system for errors and events?

### Answer

Yes, via Mailgun API for email notifications. There's a settings page where users can enable/disable notifications per event type:
1. On error occurred: enabled by default
2. On task moved to "In Review": enabled by default

## Question #6: Notification Settings Scope

Are notification settings global or per-workspace?

### Answer

Global settings with per-workspace overrides. Users can set defaults globally and then customize behavior for specific workspaces.

## Question #7: Application Interface

What's the interface for Malamar?

### Answer

A Pocketbase-like self-contained application:
1. Single executable binary (or via `bunx`/`npx`)
2. Runs as a background service
3. Uses `~/.malamar` as the directory to store database, files, etc.
4. Exposes a web interface on port 3456
5. Can be accessed remotely via Tailscale for mobile access

## Question #8: Database Choice

What database does Malamar use?

### Answer

SQLite, similar to Pocketbase. S3-compatible backup and import/export features are nice-to-haves for later consideration.

## Question #9: External Dependencies

Are there any external dependencies like Redis?

### Answer

No. Everything should be self-contained with zero external dependencies. Internal pub/sub uses an in-process event emitter. The goal is single-command execution: `./malamar`, homebrew install, or `bunx`/`npx`.

## Question #10: Working Directory Configuration

How is the working directory determined?

### Answer

Configured in workspace settings. Two scenarios:

**Scenario 1: Static Working Directory**
- User has a cloned repository with Git setup
- All agents work on that specific directory
- Everything happens there: implement, review, approve, push to remote
- User picks the directory in workspace settings

**Scenario 2: Temp Folder (Default)**
- A newly-created random working directory per task
- Ensures clean workspace for each task
- Good for multi-repo tasks (e.g., bump dependencies across repos)
- Default when creating a new workspace

## Question #11: Temp Folder Repository Context

For temp folders, how do agents know which repos to work with?

### Answer

Either in the workspace-level instruction or the task description. The user must provide repository information as part of the context since the temp folder starts empty.

## Question #12: Domain Agnostic Design

Does Malamar enforce any specific workflow like git branching strategies?

### Answer

No. Malamar is domain-agnostic - purely an orchestration layer for multi-agent CLI workflows. The agent instructions define all domain-specific behavior. Malamar only handles:
- Loop mechanics
- Response parsing

Use cases include:
- Software development
- Writing blog posts
- Browser automation via CLI
- System management
- HR spreadsheet work
- Anything achievable via AI CLIs

## Question #13: Context Passing to Agents

What context does the runner pass to an agent's CLI?

### Answer

The runner creates an input file (`/tmp/malamar_task_{task_id}.md`) containing:
1. Context about being controlled by Malamar + workspace instruction
2. Agent's instruction + brief of other agents (names only, no instructions)
3. Task details: summary, description, all comments
4. (Future P4/P5) Instructions for querying Malamar API for related data
5. Instruction to write result to output file (`/tmp/malamar_output_{random_nanoid}.json`)

The CLI is invoked with something like: "Read the $FILE_PATH and follow the instruction autonomously."

## Question #14: CLI Completion Detection

How does the runner know when the CLI has finished?

### Answer

The runner waits for the subprocess to exit. No explicit timeout configuration - let the OS handle it. However, there's a feature for users to manually kill the current loop if needed.

## Question #15: Kill Loop Behavior

What happens when the user kills a loop?

### Answer

When a user kills a loop:
1. System comment is added noting the user canceled the loop
2. Task is moved to "In Review" so the user can investigate
3. User can move the task back to "In Progress" when ready to continue

## Question #16: Task Creation UI

How does task creation work in the web UI?

### Answer

Simple form with:
- Summary (short text title)
- Description (markdown)

## Question #17: Real-Time Progress Updates

How does the user see real-time progress?

### Answer

Two mechanisms:
1. **Task-level polling (3s)**: Using React Query for gathering new task data. Works well when user is actively commenting.
2. **Global SSE endpoint**: For notifications/toasts when events happen (new comment, status change, etc.). Backend uses an in-process pub/sub to emit events.

## Question #18: Startup Recovery

When Malamar starts, what happens to tasks that were "In Progress" from a previous run?

### Answer

Auto-resume. All items in the queue are re-picked up automatically. This ensures no work is lost if Malamar is restarted.

## Question #19: Web UI Authentication

Is there authentication for the web UI?

### Answer

No authentication for now. It's meant to run locally or behind Tailscale for secure remote access without implementing a full auth system.
