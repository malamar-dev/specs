# Malamar Product Specifications

> **Malamar lets you combine the strengths of different AI CLIs into autonomous multi-agent workflows, limited only by your creativity.**

This document covers WHAT Malamar does and WHY. For implementation details (HOW), see [TECHNICAL_DESIGN.md](./TECHNICAL_DESIGN.md).

---

## Table of Contents

- [Problem Statement](#problem-statement)
- [Target Users](#target-users)
- [Application Overview](#application-overview)
- [Core Concepts](#core-concepts)
- [The Multi-Agent Loop](#the-multi-agent-loop)
- [Notifications](#notifications)
- [CLI Support](#cli-support)

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

Each AI CLI has distinct strengths:

| Tool | Strengths |
|------|-----------|
| **Claude Code** | Best-in-class instruction following, advanced agentic features (skills, session management), rich IDE integrations |
| **Gemini CLI** | Generous free tier, industry-leading context window, Google Search grounding, open-source, MCP support |
| **Codex CLI** | Session persistence/resume, built-in code review workflow, skills system, context compaction |
| **OpenCode** | Fully local/offline operation, data never leaves machine, no subscription costs, supports 75+ LLM providers |

Each tool excels in different areas. The optimal workflow combines their unique strengths for autonomous, end-to-end task completion.

### The Core Value

Malamar lets you combine the autonomous multi-agent loop with the unique strengths of different CLIs:

- **Self-correcting**: Sequential agents catch mistakes a single session would miss
- **Fresh context**: Each agent starts clean, no long-session degradation
- **Structured autonomy**: Work progresses through natural checkpoints without supervision

Even with a single CLI, the loop mechanics alone provide significant value. Add multiple CLIs, and each step can use the tool best suited for the job.

---

## Target Users

Malamar is built for:

- Those who want to combine multiple AI CLIs into one autonomous workflow
- Those who want an autonomous workflow, in the easy way
- Those who want to work with their AI agents remotely - while walking, relaxing, or just away from the laptop

---

## Application Overview

Malamar is a self-contained, Pocketbase-like application:

1. **Distribution**: Single executable binary, or run via `bunx`/`npx`. Can be installed with Homebrew.
2. **Runtime**: Runs as a background service with a single command (e.g., `./malamar`).
3. **Data Storage**: Uses a configurable data directory to store the database, files, and configuration.
4. **Web Interface**: Exposes a web UI for managing workspaces, agents, tasks, and settings.
5. **Remote Access**: Can be accessed remotely via Tailscale for mobile access when away from keyboard.
6. **Dependencies**: Zero external dependencies. Everything is self-contained (no Redis, no external database).
7. **Authentication**: No authentication for now. The application is meant to run locally or behind Tailscale.
8. **First Startup**: On first launch, Malamar creates a sample workspace with example agents so users can immediately explore how the system works.
9. **Startup Recovery**: When Malamar starts, tasks that were mid-processing are automatically re-queued and resumed. No manual intervention needed.
10. **Concurrent Access**: Multiple browser tabs or devices can access Malamar simultaneously. Changes are synchronized in real-time via polling and server-sent events.
11. **Domain Agnostic**: Malamar is purely an orchestration layer - all domain-specific behavior is defined in agent instructions. Use cases include software development, writing, browser automation, system management, and more.

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
- User specifies a directory (e.g., a cloned repository with Git setup)
- All agents work on that specific directory
- Everything happens there: implement, review, approve, and any other operations

**Temp Folder Mode (Default):**
- A new temporary directory is created per task
- Ensures a clean workspace for each task
- Good for multi-repo tasks or tasks that require isolation
- Agents can clone repositories or create files as needed
- Agents learn which repositories or resources to work with from the workspace-level instruction or task description

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
| **Skip** | Agent has nothing to do - never comment "I have nothing to do", just skip. |
| **Comment** | Agent did meaningful work. The comment contains a markdown summary of what changed. |
| **Change Status** | Agent requests to move the task to "In Review". This is the only status change agents can make. |
| **Comment + Change Status** | Agent comments and requests human attention. |

#### Workspace With No Agents

When a workspace has all its agents deleted:
- Chats can still happen with the Malamar agent (to recreate agents)
- Tasks would immediately move to "In Review" (no agents = all skip)
- A persistent warning banner is displayed until at least one agent is created

### Tasks

Tasks are work items that agents process autonomously. Each task has:

- **Summary**: Short title describing the task
- **Description**: Detailed requirements in markdown. Include all context at creation time - comments are for steering agents during processing, not adding initial context.
- **Status**: Current state (Todo, In Progress, In Review, Done)
- **Comments**: Communication channel between user, agents, and system
- **Activity Log**: History of events (creation, status changes, agent starts/finishes, comments added) - agents receive this context to understand task history

#### Task Statuses

| Status | Description |
|--------|-------------|
| **Todo** | Default when created. Waiting to be picked up by the runner. |
| **In Progress** | Agents are actively working on it. Only one task per workspace is "In Progress" at a time. |
| **In Review** | Agents agree there's nothing more to do, or human attention is needed. Waiting for user. |
| **Done** | Completed. Only humans can move tasks to Done. |

#### Task Actions

Different actions are available depending on status:

| Status | Available Actions |
|--------|-------------------|
| **Todo** | Delete, Prioritize |
| **In Progress** | Delete, Cancel, Move to In Review, Prioritize |
| **In Review** | Delete, Move to Todo |
| **Done** | Delete, Move to Todo |

**Cancel Action:** When a user cancels an "In Progress" task:
1. The CLI subprocess is killed immediately
2. Task moves to "In Review" (prevents immediate re-pickup)
3. A system comment is added: "Task cancelled by user"
4. User can investigate, fix the description/instructions, then comment to restart

#### Comments

Comments are the communication channel for all task stakeholders:
- **User comments**: Human input to steer agents
- **Agent comments**: Progress reports, questions, feedback
- **System comments**: Error notifications, status changes, cancellation notices

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

**Malamar Agent Actions:**

| Action | Description |
|--------|-------------|
| `create_agent` | Create a new agent in the workspace |
| `update_agent` | Update an existing agent's name, instruction, or CLI |
| `delete_agent` | Delete an agent from the workspace |
| `reorder_agents` | Change the execution order of agents |
| `update_workspace` | Update workspace settings |
| `rename_chat` | Rename the chat title (first response only) |

#### Workspace Agent Chat

When chatting with workspace agents (Planner, Implementer, etc.):
- **Conversational only**: Workspace agents provide advice, answer questions, and help with their domain expertise
- **No workspace actions**: Unlike the Malamar agent, workspace agents cannot create/modify agents or workspace settings
- **Rename chat**: Workspace agents can rename the chat on their first response only

#### Chat Features

**Agent Switching:**
- Users can change which agent handles the conversation mid-chat
- Full conversation history is preserved
- A system message notes the switch: "Switched from [Agent A] to [Agent B]"
- The new agent sees the entire conversation and continues with full context

**CLI Override:**
- Users can override which CLI handles the chat, regardless of the agent's default
- This is a per-chat setting that persists for all messages in that chat
- Useful for testing different CLIs or when a preferred CLI is unavailable

**File Attachments:**
- Users can upload files for agents to reference
- No file size or type restrictions - agents/CLIs handle what they can
- Duplicate filenames overwrite existing attachments
- Attachments are cleaned up when the chat is deleted

**Chat Title:**
- New chats default to "Untitled chat"
- Agents rename the chat on their first response to reflect the topic
- After the first response, agents can no longer rename the chat
- Users can manually edit the title anytime via the UI

---

## The Multi-Agent Loop

This is the core concept of Malamar - how agents work together autonomously.

### How It Works

1. **Task pickup**: The runner continuously monitors for queued tasks. Each workspace processes one task at a time.

2. **Agent execution**: For each task, the runner routes to each agent sequentially (based on agent order). Each agent:
   - Receives the full task context (summary, description, all comments, activity log)
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
- An error occurs (which triggers a retry)

### Error Handling

When errors occur (CLI failure, timeout, malformed output):
- Errors are automatically retried
- A system comment is added so agents can see what happened and self-correct in subsequent loops
- The task does NOT move to "In Review" - agents get a chance to recover

### Queue Priorities

The runner prioritizes tasks in this order:
1. Tasks explicitly prioritized by the user
2. The most recently processed task (tackle one task until completion)
3. Most recently updated tasks (LIFO - users can "bump" tasks by commenting)

Tasks in "In Review" or "Done" are not picked up for processing.

---

## Notifications

Malamar can send email notifications via Mailgun when certain events occur.

### Configurable Events

| Event | Default | Description |
|-------|---------|-------------|
| **On error occurred** | Enabled | CLI failure, timeout, or malformed output |
| **On task moved to In Review** | Enabled | Task ready for human attention |

> **Note:** Notifications are for task-related events only. Chat errors are visible immediately as system messages in the UI.

### Settings Scope

- **Global**: Mailgun configuration (API key, domain, from/to email) applies to all workspaces
- **Per-workspace**: Toggle events on/off (inherits global email config)

---

## CLI Support

Malamar supports multiple AI CLI tools through an adapter pattern.

### Supported CLIs

| CLI | Description |
|-----|-------------|
| **Claude Code** | Anthropic's CLI for Claude |
| **Gemini CLI** | Google's CLI for Gemini |
| **Codex CLI** | OpenAI's CLI for Codex |
| **OpenCode** | Open-source multi-provider CLI |

### Auto-Detection

On startup and periodically, Malamar detects available CLIs and verifies they work. The status ("Healthy" or "Unhealthy") is displayed in the CLI settings page.

### CLI Settings

Per-CLI configuration is available:
- **Binary Path**: Custom path override (empty = search in PATH)
- **Environment Variables**: Key-value pairs to inject when running the CLI
