# Technical Design: UX

This document covers user experience design decisions, detailed interaction flows, and UI mockups for Malamar.

For product specifications (what and why), see [SPECS.md](./SPECS.md).

---

## Table of Contents

- [Workspace List Page](#workspace-list-page)
- [Workspace Detail Page](#workspace-detail-page)
- [Task Detail Popup](#task-detail-popup)
- [Chat Detail Popup](#chat-detail-popup)
- [Workspace Settings Page](#workspace-settings-page)
- [Global Settings Page](#global-settings-page)

---

## Workspace List Page

The home page displays a list of workspaces as cards.

### Card Content

Each workspace card shows:
- Workspace name
- Number of agents
- Number of tasks in Todo / In Progress / In Review (Done count not shown on list)

### Sorting Options

1. **Recently active** (default): Sorted by `last_activity_at` descending
2. **Created date**: ASC or DESC by `created_at`
3. **Updated date**: ASC or DESC by `updated_at`

### Mockup

*TBD: ASCII mockup of workspace list page*

---

## Workspace Detail Page

When accessing a workspace, the user sees tabs for different views.

### Tabs

| Tab | Content |
|-----|---------|
| Tasks | Kanban board (default view) |
| Chat | List of conversations |
| Agents | Agent management |

### Tasks Tab (Kanban Board)

**Columns:**
- Four hardcoded columns: Todo, In Progress, In Review, Done
- No grouping or customization for now

**Task Ordering:**
- Within each column, tasks ordered by most recently updated on top

**Interactions:**
- Drag-and-drop between columns is NOT supported
- Status changes only happen through task detail popup
- Clicking a task opens the task detail popup

### Chat Tab

**Chat List Display:**
- Ordered by recently active (chats with recent messages at top)
- Each item shows: title, agent name, last message preview, timestamp (human diff format)

**Creating New Chats:**
- Dropdown button with options: Malamar Agent, all workspace agents

### Agents Tab

*TBD: Detailed agent management UX*

### Mockup

*TBD: ASCII mockup of workspace detail page*

---

## Task Detail Popup

Layout similar to JIRA or Linear.

### Sections

1. **Header**: Task title (editable)
2. **Description**: Markdown content (editable)
3. **Tab Section**: Switch between "Comments" and "Activity" tabs
   - On mobile: dropdown instead of tabs
   - **Comments tab**: All comments in DESC order (newest on top)
   - **Activity tab**: All activity logs in DESC order (newest on top)
4. **Action Buttons**: Based on current status (see SPECS.md for available actions)

### Mockup

*TBD: ASCII mockup of task detail popup*

---

## Chat Detail Popup

### Header

- Chat title (editable inline by clicking)
- Agent name (dropdown to change agents)
- CLI selector dropdown (next to agent selector, for CLI override)

### Agent and CLI Selection Behavior

- Agent dropdown and CLI dropdown are side by side
- Changing agent → auto-updates CLI dropdown to match agent's configured CLI
- Manually changing CLI → saves `cli_type` override to the chat
- Changing agent does NOT affect conversation history

### Agent Switch Notification

When user switches agents mid-conversation, a system message is added:
> "Changed agent from {old_agent} to {new_agent}"

**Important:** System messages NEVER trigger the agent to respond.

### Message List

- All messages displayed with oldest on top (natural reading order)
- Agent messages may include action badges/cards below the text

### Input Area

- Text input
- Attachment icon (opens file picker)
- Send/Stop button

### Processing Indicator

While processing:
- Send button changes to stop button (with pulse animation)
- User cannot send another message until processing completes
- Clicking stop cancels processing (kills CLI subprocess, marks queue item as failed)

### Mockup

*TBD: ASCII mockup of chat detail popup*

---

## Workspace Settings Page

Dedicated settings page for each workspace.

### Sections

**1. General**
- Title: The workspace name
- Description/Instruction: Context for agents

**2. Working Directory**
- Mode: Static Directory or Temp Folder (default: Temp Folder)
- Directory Path: Shown only when Static Directory mode is selected

**3. Task Cleanup**
- Auto-delete Done tasks: ON/OFF toggle (default: ON)
- Retention period: Number of days (default: 7, 0 to disable)

**4. Notifications**
- On error occurred: ON/OFF (inherits global default)
- On task moved to In Review: ON/OFF (inherits global default)

### No Agents Warning

When workspace has zero agents:
- Persistent, non-dismissable warning banner at top of workspace page
- Displayed until at least one agent is created

### Mockup

*TBD: ASCII mockup of workspace settings page*

---

## Global Settings Page

### CLI Settings Section

Per-CLI settings:
- Binary Path: Custom path override (empty = use PATH)
- Environment Variables: Key-value pairs
- Detected Version: Read-only
- Status: "Healthy" or "Unhealthy"
- Error Message: Details when unhealthy

**Refresh CLI Status Button:** For immediate re-detection after installing a new CLI.

### Mailgun Section

- API Key
- Domain
- From Email
- To Email
- **Send Test Email** button

### Notification Defaults

- On error occurred: ON/OFF (default: ON)
- On task moved to In Review: ON/OFF (default: ON)

### Mockup

*TBD: ASCII mockup of global settings page*

---

## Common Patterns

### Confirmation Dialogs

**Destructive Actions Requiring Name Confirmation:**
- Delete workspace
- Delete all done tasks

User must type the workspace name to confirm.

**Simple Confirmation:**
- Delete task
- Delete chat

Simple confirm popup.

### Real-Time Toast Notifications

Toasts shown for SSE events (see TECHNICAL_DESIGN_API.md for event types).

### CLI Unavailability Warnings

Warnings displayed on:
- Workspace detail page
- Create task form
- Task detail page

When an assigned CLI becomes unavailable.

### Mockup

*TBD: ASCII mockup of common UI patterns*