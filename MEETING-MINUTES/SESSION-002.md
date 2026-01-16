# Session 002

## CLI Integration

Q: How does Malamar invoke the CLI tools (Claude Code, Gemini CLI, Codex CLI)?

A: Direct shell invocation. For example:
```shell
claude --dangerously-skip-permissions --json-schema ... --output-format json --prompt ...
```

---

Q: For the `--json-schema` flag, is Malamar providing a schema that enforces the agent response format?

A: It depends on the CLI. Claude Code supports the json-schema flag natively. Other CLIs that don't support it will have the response format instruction embedded in the prompt.

---

Q: How does Malamar know which CLI supports what features?

A: Each supported CLI (Claude Code, Gemini CLI, OpenAI Codex CLI, OpenCode) is treated as an adapter with case-by-case hardcoded configuration for flags, command templates, timeout settings, etc. On startup, Malamar auto-detects which CLIs are available on the system.

---

Q: What happens if an agent is assigned to a CLI that isn't available?

A: The assignment is allowed, but at runtime if the CLI isn't available, the runner treats it as an error - comments the error into the task and the standard error handling flow applies (system comment, new queue item, retry via queue).

---

## Notifications

Q: Is there a notification system for errors and events?

A: Yes, via Mailgun API for email notifications. There's a settings page where users can enable/disable notifications per event type:
1. On error occurred: enabled by default
2. On task moved to in review: enabled by default

---

Q: Are notification settings global or per-workspace?

A: Global settings with per-workspace overrides.

---

## Application Architecture

Q: What's the interface for Malamar?

A: A Pocketbase-like self-contained application:
1. Single executable binary (or via bunx/npx)
2. Runs as a background service
3. Uses `~/.malamar` as directory to store database, files, etc.
4. Exposes a web interface on port 3456
5. Can be accessed remotely via Tailscale for mobile access

---

Q: What database does Malamar use?

A: SQLite (like Pocketbase). S3-compatible backup and import/export features are nice-to-haves for later.

---

Q: Are there any external dependencies like Redis?

A: No. Everything should be self-contained with zero external dependencies. Internal pub/sub uses an in-process event emitter. The goal is single-command execution: `./malamar`, homebrew install, or `bunx/npx`.

---

## Working Directory

Q: How is the working directory determined?

A: Configured in workspace settings. Two scenarios:

**Scenario 1: Static Working Directory**
- User has a cloned repository with Git setup
- All agents work on that specific directory
- Everything happens there: implement, review, approve, push to remote
- User picks the directory in workspace settings

**Scenario 2: Temp Folder (Default)**
- A newly-created random working directory per task
- Ensures clean workspace
- Good for multi-repo tasks (e.g., bump dependencies across repos)
- Default when creating a new workspace

---

Q: For temp folders, how do agents know which repos to work with?

A: Either in the workspace-level instruction or the task description.

---

## Domain Agnostic Design

Q: Does Malamar enforce any specific workflow like git branching strategies?

A: No. Malamar is domain-agnostic - purely an orchestration layer for multi-agent CLI workflows. The agent instructions define all domain-specific behavior. Malamar only handles loop mechanics and response parsing. Use cases include:
- Software development
- Writing blog posts
- Browser automation via CLI
- System management
- HR spreadsheet work
- Anything achievable via AI CLIs

---

## Context Passing

Q: What context does the runner pass to an agent's CLI?

A: The runner creates an input file (`/tmp/malamar_task_{task_id}.md`) containing:
1. Context about being controlled by Malamar + workspace instruction
2. Agent's instruction + brief of other agents (names only, no instructions)
3. Task details: summary, description, all comments
4. (Future P4/P5) Instructions for querying Malamar API for related data
5. Instruction to write result to output file (`/tmp/malamar_output_{random_nanoid}.json`)

The CLI is invoked with something like: "Read the $FILE_PATH and follow the instruction autonomously."

---

Q: How does the runner know when the CLI has finished?

A: Waits for the subprocess to exit. No explicit timeout configuration - let the OS handle it. But there's a feature for users to manually kill the current loop.

---

Q: What happens when the user kills a loop?

A: System comment noting the user canceled the loop, then move the task to "In Review" so the user can investigate. User can move it back to "In Progress" when ready.

---

## Web UI

Q: How does task creation work in the web UI?

A: Simple form with:
- Summary (short text title)
- Description (markdown)

---

Q: How does the user see real-time progress?

A: Two mechanisms:
1. **Task-level polling (3s)**: Using React Query for gathering new task data. Works well when user is actively commenting.
2. **Global SSE endpoint**: For notifications/toasts when events happen (new comment, status change, etc.). Backend uses an in-process pub/sub to emit events.

---

## Startup and Recovery

Q: When Malamar starts, what happens to tasks that were "In Progress" from a previous run?

A: Auto-resume. All items in the queue are re-picked up automatically.

---

## Authentication

Q: Is there authentication for the web UI?

A: No authentication for now. It's meant to run locally or behind Tailscale.
