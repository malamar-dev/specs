# Session 009

Please read [SKILL.md](../.claude/skills/explore-specs/SKILL.md) to understand the process used to create this file.

**Session Focus:** Clarifying document organization between SPECS.md and TECHNICAL_DESIGN.md, addressing formatting issues, fragmented knowledge, duplication, and filling content gaps before implementation.

---

## Question #1: Document Organization Principle

Before reorganizing content, we need clear principles for what belongs where. Currently, both SPECS.md and TECHNICAL_DESIGN.md contain configuration tables (duplicated), tech stack information, and CLI commands.

SPECS.md says it covers "WHAT and WHY" while TECHNICAL_DESIGN.md covers "HOW" - but the current content doesn't follow this strictly.

What criteria should we use to decide where content belongs?

### Answer

**User-observable vs Internal**: Content that defines user-facing behavior (features, capabilities, data users see/interact with) goes in SPECS. Content about internal implementation (algorithms, file formats, tech choices) goes in TECHNICAL_DESIGN.

---

## Question #2: Handling Configuration Parameters

The configuration tables (host, port, data directory, logging, runner poll interval, temp files) are currently duplicated in both README.md and TECHNICAL_DESIGN.md.

Where should configuration parameters live?

### Answer

**SPECS mentions, TECHNICAL_DESIGN details**: SPECS mentions that configuration exists and highlights the most commonly used options as examples, then redirects readers to TECHNICAL_DESIGN for full details. TECHNICAL_DESIGN contains the complete configuration reference with all parameters, defaults, env var names, CLI flags, and override behavior.

---

## Question #3: README's Role and Content

Currently README.md contains project introduction, documentation navigation, full tech stack section, full configuration tables, and CLI commands table. Given the document is for a specs repository (not the code repository), what should README contain?

### Answer

**Navigation hub only**: README should explain that this is the specifications repository for Malamar (the actual code lives in a separate repository) and provide navigation to the various docs. Operational content like install instructions, quick start, how to run - all of that belongs in the code repository's README, not here. Tech stack, configuration tables, CLI commands - these will live in the code repo's docs.

---

## Question #4: The Non-Existent TECHNICAL_DESIGN Files

The current README references files that don't exist: TECHNICAL_DESIGN_DATA.md, TECHNICAL_DESIGN_RUNTIME.md, TECHNICAL_DESIGN_API.md, TECHNICAL_DESIGN_UX.md, TECHNICAL_DESIGN_UI.md. Meanwhile, all content is in a single TECHNICAL_DESIGN.md (677 lines).

Should we split TECHNICAL_DESIGN into multiple files?

### Answer

**Keep monolithic**: TECHNICAL_DESIGN.md stays as a single file. Splitting creates too much friction. The README references to non-existent split files need to be removed. Good headings and table of contents within the single file will provide navigation.

---

## Question #5: Data Model / Entity Schema Placement

Entity schemas appear in both docs - SPECS.md describes entities conceptually while TECHNICAL_DESIGN.md shows database schemas with field types.

How should we handle entity/data model documentation?

### Answer

**Concept in SPECS, schema in TECHNICAL_DESIGN**: SPECS describes what entities are and what they mean in prose (workspace has agents and tasks). TECHNICAL_DESIGN shows the exact database schema (field names, types, constraints). No field lists in SPECS - just conceptual explanations.

---

## Question #6: The Multi-Agent Loop - What vs How

The multi-agent loop is Malamar's core concept and appears in both docs with significant overlap. Both explain "how the loop works" but at different levels of detail.

Where does the loop belong?

### Answer

**Behavioral contract in SPECS, algorithm in TECHNICAL_DESIGN**: SPECS explains what users can expect (agents run in order, loop continues until done, errors retry automatically). TECHNICAL_DESIGN explains how it's implemented (worker concurrency, queue pickup SQL, JIT snapshots).

---

## Question #7: API Endpoints and SSE Events

TECHNICAL_DESIGN.md contains full API documentation including REST endpoints and SSE event types with payload formats.

Where should API documentation live?

### Answer

**All in TECHNICAL_DESIGN**: API endpoints and SSE formats are implementation details. The web UI abstracts these away from users. Developers/integrators read TECHNICAL_DESIGN. SPECS only mentions "real-time updates via SSE" conceptually without detailing endpoints or event formats.

---

## Question #8: CLI Adapter Details and Invocation

TECHNICAL_DESIGN.md contains detailed CLI adapter information including command templates, response format enforcement, and input/output file formats.

Where should CLI adapter documentation live?

### Answer

**Split: Context format in SPECS, commands in TECHNICAL_DESIGN**: SPECS explains what context agents receive (workspace instruction, agent instruction, task summary, comments, activity log) so users can write effective instructions. TECHNICAL_DESIGN shows the exact file paths, command templates, markdown structure, and JSON schemas.

---

## Question #9: Chat Feature Documentation

The Chat feature has similar documentation split concerns with input/output file formats, context files, attachments, and working directory behavior.

Should Chat follow the same pattern as task context?

### Answer

**Yes, same pattern**: SPECS explains what chat agents can access (conversation history, workspace state, attached files, working directory based on workspace mode). TECHNICAL_DESIGN shows exact file formats, paths, and queue processing mechanics.

---

## Question #10: Default Values and Magic Numbers

Throughout the docs, there are various default values (7-day retention, 3s polling, 1000ms runner interval, 5-minute health check, port 3456).

How should we document default values?

### Answer

**Configurable defaults in TECHNICAL_DESIGN only**: Since configuration details go in TECHNICAL_DESIGN, all defaults live there. SPECS mentions that things are "configurable" without specifying exact values (e.g., "done tasks can be auto-deleted after a configurable retention period").

---

## Question #11: Future Vision Section

SPECS.md currently ends with a "Future Vision" section listing potential future enhancements (S3 Backup, Import/Export, etc.).

Should this stay in SPECS?

### Answer

**Move to a separate ROADMAP.md**: Future vision is valuable but distinct from specifications. A dedicated file makes it clear these are ideas, not commitments. SPECS stays focused on the current product.

---

## Question #12: Tech Stack Placement

Tech stack information currently appears in README.md and TECHNICAL_DESIGN.md.

Since this is a specs repository (not the code repo), where should tech stack live?

### Answer

**TECHNICAL_DESIGN.md only**: Tech stack is an implementation decision that constrains how Malamar is built. It belongs in the technical design document. Remove from README (which becomes navigation-only).

---

## Question #13: Notifications Documentation

Notifications (Mailgun email) currently appear in both docs with user-facing behavior in SPECS and implementation details in TECHNICAL_DESIGN.

Is the current split correct?

### Answer

**Current split is correct**: SPECS covers what notifications exist and when they fire (user-observable). TECHNICAL_DESIGN covers how they're sent (fire-and-forget, failure handling). No changes needed.

---

## Question #14: Error Handling Documentation

Error handling appears in multiple places with nearly duplicate content in both SPECS and TECHNICAL_DESIGN.

How should we handle error handling documentation?

### Answer

**Behavioral outcome in SPECS, mechanics in TECHNICAL_DESIGN**: SPECS says "errors are automatically retried; a system comment is added so agents can see what happened and self-correct in subsequent loops." TECHNICAL_DESIGN explains the queue-based retry mechanism, no explicit backoff, and how system comments trigger new queue items.

---

## Question #15: CLI Commands Documentation

The CLI commands table appears identically in both README.md and TECHNICAL_DESIGN.md.

Where should CLI commands live?

### Answer

**TECHNICAL_DESIGN only**: CLI commands are operational details fitting under the "Operations" section of TECHNICAL_DESIGN. README becomes navigation-only. The actual Malamar code repo's README will have these commands for users.

---

## Question #16: Working Directory Documentation

Working directory modes (Static vs Temp Folder) appear in both docs, with SPECS being conceptual and TECHNICAL_DESIGN covering chat-specific implementation. But TECHNICAL_DESIGN doesn't fully document working directory handling for tasks.

Should we expand the documentation?

### Answer

**Keep SPECS concept, expand TECHNICAL_DESIGN implementation**: SPECS explains what working directory modes mean and when to use each. TECHNICAL_DESIGN should document how working directories are used for both tasks AND chat (path patterns, CLI invocation with cwd).

---

## Question #17: User Interface Sections in Both Docs

Both documents have "User Interface" sections with some overlap in real-time updates and page-level details in SPECS.

Should SPECS contain UI page descriptions?

### Answer

**Remove page-level UI from SPECS**: SPECS describes capabilities (manage workspaces, view tasks in kanban, chat with agents) without listing specific pages. TECHNICAL_DESIGN covers page structure, UI implementation, and navigation patterns.

---

## Question #18: The Malamar Agent Documentation

The Malamar Agent is a special built-in agent with specific capabilities. It's both a user-facing feature AND has technical implementation.

Where should Malamar Agent documentation live?

### Answer

**Capabilities in SPECS, implementation in TECHNICAL_DESIGN**: SPECS explains what the Malamar Agent can do and when to use it. TECHNICAL_DESIGN explains how it's implemented (knowledge base access, action execution, context file structure). AGENTS/ folder contains the actual knowledge base content.

---

## Question #19: CLI Support Documentation

SPECS.md has a "CLI Support" section mixing user-observable info (supported CLIs) with implementation details (binary names, auto-detection behavior).

How should this be organized?

### Answer

**Supported CLIs in SPECS, detection mechanics in TECHNICAL_DESIGN**: SPECS lists which CLIs are supported and that custom paths and env vars are configurable. TECHNICAL_DESIGN explains binary names, auto-detection algorithm, health check interval, and test prompt behavior.

---

## Question #20: Activity Logs Documentation

Activity logs are user-observable but the current SPECS doesn't really explain them.

Is there a gap in SPECS?

### Answer

**Add activity log concept to SPECS**: SPECS should explain that tasks have activity logs showing history (creation, status changes, agent starts/finishes, comments) and that agents receive this context. TECHNICAL_DESIGN keeps the schema details, JSONL format, and API endpoint.

---

## Question #21: SPECS.md Internal Structure

With all the changes decided, what should the SPECS structure look like after reorganization?

### Answer

**Streamlined feature-focused structure:**

1. Problem Statement (why Malamar exists)
2. Target Users
3. Core Concepts (Workspaces, Agents, Tasks, Chat, Activity Logs)
4. The Multi-Agent Loop (behavioral contract)
5. Notifications
6. CLI Support (which CLIs, configurable)
7. Application Overview (self-contained, local-first, remote access via Tailscale)

---

## Question #22: TECHNICAL_DESIGN.md Internal Structure

What should the TECHNICAL_DESIGN structure look like?

### Answer

**Implementation-domain structure:**

1. Architecture Overview (tech stack, high-level system design)
2. Data Model (schemas, ID generation, cascade deletes)
3. Runner & Queue Mechanics (runner logic, queue pickup, loop flow, error handling)
4. CLI Adapters (command templates, context passing for tasks and chat)
5. API & Real-Time (REST endpoints, SSE events)
6. User Interface Implementation (page structure, layouts, components, forms, interactions)
7. Background Jobs (cleanup, health check)
8. Configuration Reference (all settings, defaults, env vars, CLI flags)
9. Operations (CLI commands, distribution/packaging)

---

## Question #23: README.md Structure

What should README contain for a specs repository?

### Answer

**Minimal navigation:**

1. Project tagline
2. Brief explanation: "This is the specification repository for Malamar. The actual code lives at [link]."
3. Documentation index:
   - SPECS.md - What Malamar does and why
   - TECHNICAL_DESIGN.md - How it's implemented
   - ROADMAP.md - Future vision
   - AGENTS/ - Knowledge base for the Malamar agent

---

## Question #24: ROADMAP.md Structure

What should the new ROADMAP.md file contain?

### Answer

**Simple ideas list**: Title + brief description for each idea. No priorities, no timelines - those change too often. Keep it lightweight and low-maintenance.

---

## Question #25: Content Gaps - Missing Specifications

Are there gaps - things that should be specified but aren't in either document?

### Answer

Multiple gaps were identified and addressed in subsequent questions:
- Task cancellation behavior
- First startup / empty state
- Database migrations / versioning
- Concurrent access behavior
- CLI becomes unhealthy mid-task
- Workspace deletion while processing
- Task deletion while in progress
- Startup recovery behavior
- Malamar Agent actions
- Workspace agent chat actions
- Chat agent switching
- Chat CLI override
- File attachment behavior
- Chat title behavior

---

## Question #26: Task Cancellation Behavior

When a user clicks "Cancel" on an "In Progress" task, what exactly should happen?

### Answer

**Kill process, move to In Review, add system comment:**

1. CLI subprocess is killed immediately
2. Task moves to "In Review" (prevents immediate re-pickup)
3. System comment added: "Task cancelled by user"
4. User investigates and fixes description/instructions
5. When user comments, task moves back to Todo → In Progress
6. User can prioritize if immediate processing needed

This gives the user control to investigate before agents touch the task again.

---

## Question #27: First Startup / Empty State

When Malamar starts for the first time (fresh install, empty database), what happens?

### Answer

**Auto-create sample workspace**: A demo workspace is created automatically with sample agents so users can immediately see how the system works. Can be deleted when no longer needed.

The sample workspace is a **software development example**:
- Title: "Sample: Code Assistant" (or similar)
- 4 agents: Planner, Implementer, Reviewer, Approver
- Each with generic but useful instructions demonstrating the workflow
- No sample tasks - user creates their first task

---

## Question #28: Database Migrations / Versioning

How are database migrations handled when Malamar updates?

### Answer

**Automatic with panic on failure:**

- Migrations run automatically on startup
- Application panics on failure to ensure data consistency
- Users should backup `~/.malamar` before major updates (document this recommendation)
- No automatic rollback - manual restore from backup if needed
- Simple approach appropriate for local-first single-user app

---

## Question #29: Concurrent Access Behavior

What happens when multiple clients access Malamar simultaneously (multiple browser tabs, mobile + desktop via Tailscale)?

### Answer

**Last-write-wins, UI stays synced:**

- No locking mechanism
- Multiple tabs can edit - last save wins
- SSE and polling keep all tabs synchronized
- If user comments while agent is processing, comment is added and agent continues (sees it on next loop iteration)
- Simple approach matching single-user assumption

---

## Question #30: CLI Becomes Unhealthy Mid-Task

What happens if a CLI that an agent uses becomes unhealthy while a task is being processed?

### Answer

**No pre-check, natural failure handling:**

- Don't check CLI health before invocation - just try to run it
- If CLI fails, normal error handling kicks in (system comment, retry via queue)
- If CLI stays unhealthy, task keeps retrying with accumulating error comments
- User sees errors and can fix the CLI or reassign agent to a different CLI

---

## Question #31: Workspace Deletion While Task Processing

What happens if a user deletes a workspace while a task is actively being processed?

### Answer

**Kill subprocess, then cascade delete:**

- If any task in the workspace is being processed, kill the CLI subprocess first
- Then proceed with cascade delete
- Clean shutdown - no orphaned processes
- Same logic applies to chat processing - kill any active chat CLI subprocess before deleting

---

## Question #32: Task Deletion While In Progress

What if a user deletes a task that's currently being processed?

### Answer

**Allow deletion with subprocess kill:**

- Delete action IS available for In Progress tasks (updating the earlier SPECS table)
- Kill subprocess first, then cascade delete
- One-click removal for stuck or problematic tasks

This updates the Task Actions table - Delete is now available for all statuses.

---

## Question #33: Startup Recovery Behavior

What happens to tasks that were mid-CLI-execution when Malamar crashed?

### Answer

**Reset in_progress queue items, resume naturally:**

- On startup, find any queue items (task or chat) with status "in_progress"
- Reset them to "queued"
- Runner picks them up naturally
- Tasks stay "In Progress" status, just re-queued for processing
- Clean crash recovery without special logic

---

## Question #34: Malamar Agent Actions

What actions should the Malamar Agent be able to perform?

### Answer

**Workspace management actions only:**

- `create_agent` - Create a new agent
- `update_agent` - Update agent name/instruction/CLI
- `delete_agent` - Delete an agent
- `reorder_agents` - Change agent execution order
- `update_workspace` - Update workspace settings
- `rename_chat` - Rename the chat (first response only)

No `create_task` - tasks should be created explicitly via UI, not through chat conversation. No destructive workspace-level actions (delete workspace, delete all tasks) - those require explicit UI actions.

---

## Question #35: Workspace Agent Chat Actions

Should workspace agents (Planner, Implementer, etc.) in chat also have actions?

### Answer

**Malamar Agent has management actions, workspace agents are conversational only (with one exception):**

- **Malamar Agent**: Can perform workspace management actions (create/update/delete/reorder agents, update workspace) always available, plus `rename_chat` on first response only
- **Workspace agents**: `rename_chat` on first response only; otherwise conversational only - provide advice, answer questions, help with domain expertise
- **No agent can create tasks via chat** - tasks must be created through the UI explicitly

---

## Question #36: Chat Agent Switching Behavior

When switching agents mid-chat, what happens to conversation history?

### Answer

**Full history preserved, system message added:**

- Full conversation history is preserved when switching
- System message added: "Switched from [Agent A] to [Agent B]"
- New agent sees entire conversation and continues with full context

---

## Question #37: Chat CLI Override Behavior

How does CLI override work in chat?

### Answer

**Per-chat setting, Malamar Agent uses system default:**

- CLI override is a per-chat setting (not per-message)
- Once set, all messages in that chat use the override CLI
- For Malamar Agent (or when no override set), use first healthy CLI in this hardcoded order:
  1. Claude Code
  2. Codex CLI
  3. Gemini CLI
  4. OpenCode
- UI: dropdown in chat header or settings

---

## Question #38: File Attachment Behavior

What are the limits and behavior for file attachments in chat?

### Answer

**No restrictions:**

- No file size limit
- No file type restrictions (agents/CLIs handle what they can)
- Duplicate filenames: overwrite existing
- No explicit "remove attachment" - attachments cleaned up when chat is deleted
- CLIs that support images/binaries can use them; others treat as text

---

## Question #39: Chat Title Behavior

How are chat titles managed?

### Answer

**Agent renames on first response, user can edit anytime:**

All agents (Malamar AND workspace agents) have the `rename_chat` action, but it's ONLY available on the first agent response.

**Chat title flow:**
1. User creates chat → defaults to "Untitled chat"
2. User sends message
3. Agent responds + renames chat to match the topic (via `rename_chat` action)
4. After first response, `rename_chat` is no longer available to agents
5. User can manually edit title anytime via UI

Agent instructions should instruct the agent to update the chat name in their first response to reflect the topic the user is discussing.
