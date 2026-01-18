# Malamar Product Specifications

> **Malamar lets you combine the strengths of different AI CLIs into autonomous multi-agent workflows, limited only by your creativity.**

This document covers WHAT Malamar does and WHY. For implementation details (HOW), see the technical design files linked in [README.md](./README.md).

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [The Vision](#the-vision)
- [Target Users](#target-users)
- [Application Overview](#application-overview)
- [Core Concepts](#core-concepts)
- [The Multi-Agent Loop](#the-multi-agent-loop)
- [User Interface](#user-interface)
- [Notifications](#notifications)
- [CLI Support](#cli-support)
- [Future Vision](#future-vision)

---

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

---

## The Vision

With Malamar, the workflow becomes:

1. Set up a workspace with agents configured to your best practices
2. Create a task with detailed requirements
3. Walk away
4. Come back later - everything is done, automatically

Human intervention happens through the "In Review" status, email notifications, and the ability to steer agents via comments - even remotely from a phone via Tailscale.

---

## Target Users

Malamar is built for developers and power users who:
- Use multiple AI CLI tools and want to orchestrate them
- Want autonomous task completion without constant supervision
- Need flexibility to define their own workflows via agent instructions

---

## Application Overview

Malamar is a self-contained, Pocketbase-like application:

1. **Distribution**: Single executable binary, or run via `bunx`/`npx`. Can be installed with Homebrew.
2. **Runtime**: Runs as a background service with a single command (e.g., `./malamar`).
3. **Data Storage**: Uses `~/.malamar` directory to store the SQLite database, files, and configuration.
4. **Web Interface**: Exposes a web UI on port `3456` for managing workspaces, agents, tasks, and settings.
5. **Remote Access**: Can be accessed remotely via Tailscale for mobile access when away from keyboard.
6. **Dependencies**: Zero external dependencies. Everything is self-contained (no Redis, no external database).
7. **Authentication**: No authentication for now. The application is meant to run locally or behind Tailscale.

**Startup and Recovery:** When Malamar starts, all items in the task event queue are automatically picked up and resumed. No manual intervention is needed for tasks that were "In Progress" from a previous run.

**Domain Agnostic:** Malamar is purely an orchestration layer. It has no opinions on what agents do - all domain-specific behavior is defined in agent instructions. Use cases include software development, writing blog posts, browser automation, system management, HR spreadsheet work, and more.

---

## Core Concepts

### Workspaces

A workspace is the roof of everything in Malamar. It contains:

- **Title**: For the user to identify the workspace
- **Description/Instruction**: Context for agents to understand the overall purpose
- **Agents**: The AI agents that work on tasks (typically: Planner, Implementer, Reviewer, Approver)
- **Tasks**: Work items that agents process autonomously
- **Chats**: Conversations with agents for ad-hoc assistance

#### Working Directory

The working directory determines where agents execute their work. Two modes:

**Static Directory Mode:**
- User has a pre-configured directory (e.g., a cloned repository with Git setup)
- All agents work on that specific directory
- Everything happens there: implement, review, approve, and any other operations

**Temp Folder Mode (Default):**
- A newly-created random working directory per task (`/tmp/malamar_tasks_{task_id}`)
- Ensures a clean workspace for each task
- Good for multi-repo tasks or tasks that require isolation
- Agents can clone repositories or create files as needed

For temp folder scenarios, agents learn which repositories or resources to work with from the workspace-level instruction or the task description.

**Working Directory Conflicts:** Malamar does not prevent multiple workspaces from using the same static directory path. Users are responsible for avoiding conflicts.

**Mid-Flight Changes:** If the working directory is changed while a task is actively being processed, the currently running agent continues with the old directory. Subsequent agents pick up the new path.

### Agents

Agents are AI-powered workers that process tasks. Each agent has:

- **Name**: Human-readable identifier (e.g., "Planner", "Implementer")
- **Instruction**: System prompt defining the agent's role, responsibilities, and behavior
- **CLI Assignment**: Which AI CLI tool the agent uses (Claude Code, Gemini CLI, etc.)
- **Order**: Execution sequence within the workspace

#### Example Agent Roles

| Role | Responsibility |
|------|----------------|
| **Planner** | Enrich task requirements, research context, create implementation plan. Comment questions and move to "In Review" if requirements are unclear. |
| **Implementer** | Execute the plan, do the actual work. Verify and implement reviewer feedback. |
| **Reviewer** | Review work quality, ensure it meets standards. Discuss with implementer via comments until work is ready. |
| **Approver** | Final verification against requirements. Move to "In Review" when ready for human approval. |

#### Agent Instructions Responsibility

Writing effective agent instructions is the **user's responsibility**. The user must define:
- **WHEN** to skip, comment, or move a task to "In Review"
- **WHAT** the agent should focus on and verify

Malamar's responsibility is the **HOW**:
- Ensuring agents respond in the correct format
- Executing the actions agents return
- Managing the loop and queue mechanics

#### Agent Actions

When an agent finishes working on a task, it can take these actions:

| Action | Description |
|--------|-------------|
| **Skip** | Agent has nothing to do. Use when the task doesn't require this agent's expertise, another agent should handle it, or the agent has nothing meaningful to contribute. |
| **Comment** | Agent did meaningful work and has progress to report. The comment contains a markdown summary of what changed. |
| **Change Status** | Agent requests to move the task to "In Review". This is the only status change agents can make. |

**Important:** An agent should NEVER comment "I have nothing to do" or similar. If there's nothing to do, use skip.

**Valid Combinations:**
1. Skip alone - agent has nothing to do
2. Comment alone - agent did work, loop continues
3. Comment + change status to "In Review" - agent comments and requests human attention
4. Change status alone - technically valid but discouraged; agents should always explain why

#### Workspace With No Agents

When a workspace has all its agents deleted:
- Chats can still happen with the Malamar agent (to recreate agents)
- Tasks would immediately move to "In Review" (no agents = all skip)
- A persistent warning banner is displayed until at least one agent is created

### Tasks

Tasks are work items that agents process autonomously. Each task has:

- **Summary**: Short title describing the task
- **Description**: Detailed requirements in markdown (all context must be here at creation time)
- **Status**: Current state (Todo, In Progress, In Review, Done)
- **Comments**: Communication channel between user, agents, and system

**Important:** When creating a task, users must include all context in the description. Comments are for steering agents during processing, not for adding initial context. The runner may pick up a task immediately after creation.

#### Task Statuses

| Status | Description |
|--------|-------------|
| **Todo** | Default when created. Waiting to be picked up by the runner. |
| **In Progress** | Agents are actively working on it. Only one task per workspace is "In Progress" at a time. |
| **In Review** | Agents agree there's nothing more to do, or human attention is needed. Waiting for user. |
| **Done** | Completed. Only humans can move tasks to Done. |

#### Comments

Comments are the communication channel for all task stakeholders:
- **User comments**: Human input to steer agents
- **Agent comments**: Progress reports, questions, feedback
- **System comments**: Error notifications, status changes

### Chat

The Chat feature allows users to have conversations with agents for ad-hoc assistance:

- **Workspace agents**: Chat with any agent for help related to their expertise
- **Malamar agent**: A special built-in agent for managing the workspace, tuning agents, and answering Malamar questions

#### Malamar Agent

The Malamar agent is a special built-in agent that helps users:
- Create, update, delete, and reorder agents
- Update workspace settings
- Answer questions about how to use Malamar
- Help write effective agent instructions

The Malamar agent can perform administrative actions on the workspace (create agents, update settings, create tasks) but cannot delete the workspace itself - that requires explicit human confirmation.

#### Chat Features

- **Agent switching**: Change which agent handles the conversation mid-chat
- **CLI override**: Override which CLI handles the chat (regardless of agent's default)
- **File attachments**: Upload files for agents to reference
- **Actions**: Agents can perform actions (create tasks, create agents) alongside their responses

---

## The Multi-Agent Loop

This is the core concept of Malamar - how agents work together autonomously.

### How It Works

1. **Task pickup**: The runner continuously monitors for queued tasks. Each workspace processes one task at a time.

2. **Agent execution**: For each task, the runner routes to each agent sequentially (based on agent order). Each agent:
   - Receives the full task context (summary, description, all comments)
   - Works on the task using their assigned CLI
   - Responds with actions (skip, comment, and/or change status)

3. **Loop continuation**: After all agents finish one pass:
   - If any agent added a comment → retrigger the loop from the first agent
   - If all agents skipped → move task to "In Review"
   - If any agent requested "In Review" → stop immediately, move task to "In Review"

4. **Human intervention**: Tasks in "In Review" wait for user input. When a user comments, the task moves back to "In Progress" and the loop restarts.

### Iteration Limits

There is no hard cap on iteration count or time. The loop continues until:
- All agents skip (nothing to do), OR
- An agent requests "In Review", OR
- An error occurs (which triggers a retry via the queue)

### Error Handling

When errors occur (CLI failure, timeout, malformed output):
1. A System comment is added with error details
2. The current loop stops
3. A new queue item is created for retry
4. The task does NOT move to "In Review" - agents can self-correct in subsequent loops

### Queue Priorities

The runner prioritizes tasks in this order:
1. Tasks explicitly prioritized by the user
2. The most recently processed task (tackle one task until completion)
3. Most recently updated tasks (LIFO - users can "bump" tasks by commenting)

Tasks in "In Review" or "Done" are not picked up for processing.

---

## User Interface

The web interface is exposed on port `3456`. This section covers high-level concepts; detailed UI/UX specifications are in [TECHNICAL_DESIGN_UX.md](./TECHNICAL_DESIGN_UX.md).

### Pages Overview

| Page | Purpose |
|------|---------|
| **Workspace List** | Home page showing all workspaces as cards with activity summaries |
| **Workspace Detail** | Tabs for Tasks (Kanban), Chat, and Agents |
| **Workspace Settings** | Configure working directory, cleanup, notifications |
| **Task Detail** | View/edit task, comments, activity logs, take actions |
| **Chat Detail** | Conversation with an agent, file attachments |
| **Global Settings** | CLI configuration, Mailgun setup |

### Task Kanban Board

Tasks are displayed in a Kanban board with four columns: Todo, In Progress, In Review, Done.
- Tasks are ordered by most recently updated
- Clicking a task opens the task detail popup
- No drag-and-drop between columns (status changes happen in task detail)

### Task Actions

Different actions are available depending on status:

| Status | Available Actions |
|--------|-------------------|
| **Todo** | Delete, Prioritize |
| **In Progress** | Cancel (stops current loop), Move to In Review, Prioritize |
| **In Review** | Move to Todo, Delete |
| **Done** | Move to Todo, Delete |

**Delete All Done Tasks:** Users can bulk-delete all completed tasks in a workspace (requires confirmation by typing workspace name).

### Real-Time Updates

The UI stays in sync through:
- Polling every 3 seconds for active views
- Server-sent events (SSE) for toast notifications when things happen

---

## Notifications

Malamar can send email notifications via Mailgun when certain events occur.

### Configurable Events

| Event | Default | Description |
|-------|---------|-------------|
| **On error occurred** | Enabled | CLI failure, timeout, or malformed output |
| **On task moved to In Review** | Enabled | Task ready for human attention |

**Note:** Notifications are for task-related events only. Chat errors are visible immediately as system messages in the UI.

### Settings Scope

- **Global**: Mailgun configuration (API key, domain, from/to email) applies to all workspaces
- **Per-workspace**: Toggle events on/off (inherits global email config)

---

## CLI Support

Malamar supports multiple AI CLI tools through an adapter pattern.

### Supported CLIs

| CLI | Binary Name |
|-----|-------------|
| Claude Code | `claude` |
| Gemini CLI | `gemini` |
| OpenAI Codex CLI | `codex` |
| OpenCode | `opencode` |

### Auto-Detection

On startup and periodically, Malamar automatically detects available CLIs:
1. Search for the binary in PATH (or custom path from settings)
2. Run a simple test prompt to verify it works
3. Store the status ("Healthy" or "Unhealthy")

### CLI Settings

Per-CLI configuration:
- **Binary Path**: Custom path override (empty = use PATH)
- **Environment Variables**: Key-value pairs to inject when running the CLI

**CLI Unavailability:** Agent assignment to a CLI is allowed even if the CLI isn't currently available. At runtime, if an assigned CLI isn't available, it's treated as an error (system comment added, retry via queue).

**Unavailability Warnings:** Warnings are displayed on workspace detail, create task form, and task detail pages when an assigned CLI becomes unavailable.

---

## Future Vision

Future enhancements to consider (not for initial implementation):

### S3 Backup

S3-compatible automatic backup for the SQLite database and files in `~/.malamar`.

### Import/Export

Ability to export workspaces, tasks, and settings for backup or migration, and import them into another Malamar instance.

### Related Tasks Query

Allow agents to query Malamar's API to collect additional information such as related tasks in the same workspace.

### In-Browser Push Notifications

Add in-browser push notifications as a notification channel, using the same configurable events as email notifications.

### Clone Workspace

Ability to clone a workspace, including its settings and agents, but starting fresh with no tasks. May extend to cloning individual agents or tasks.

### Agent Client Protocol (ACP)

Investigate using the Agent Client Protocol (ACP, https://zed.dev/acp) as an alternative to calling AI CLIs via subprocess commands.

### Community Gallery

A gallery where users can share and discover agent configurations created by the community.