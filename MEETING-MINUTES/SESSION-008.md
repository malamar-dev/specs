# Session 008

Please read [SKILL.md](../.claude/skills/explore-specs/SKILL.md) to understand the process used to create this file.

## Question #1: Chat Processing Timeout

For chat processing, is there any timeout consideration, or is it purely user-initiated cancellation like tasks?

### Answer

No timeout consideration for chat processing, purely user-initiated cancellation via the stop button.

## Question #2: Remote Knowledge Base Failures

When the Malamar agent fetches the remote knowledge base URL and the network is unavailable or the URL returns an error, should this be treated as a silent failure or surface as a system message?

### Answer

Should be treated as a silent failure. The Malamar agent proceeds without the knowledge base if unreachable.

## Question #3: Reorder Agents Validation

For the `reorder_agents` action, what happens if the array is incomplete (missing some agent IDs) or contains invalid IDs?

### Answer

Reject the entire action with a system message. The action must include all valid agent IDs in the workspace.

## Question #4: Mid-Flight Working Directory Changes

If the working directory is changed (via Malamar agent action or manual UI) while a task is actively being processed, what happens?

### Answer

Accept the risk. The currently running agent continues with the old directory; subsequent agents pick up the new path. This applies to both Malamar agent actions and manual user changes.

## Question #5: Agent Switch System Message

When a user switches agents mid-conversation in a chat, should the input file indicate the context shift?

### Answer

Yes, add a new system message like "Changed agent from Planner to Malamar" when the user switches agents. Important: Only user messages trigger CLI responses - system messages NEVER trigger the agent to respond.

## Question #6: Create Task Immediate Processing

When an agent uses the `create_task` action from chat, should there be a delay or draft state before runner pickup?

### Answer

No, immediate processing is expected. The agent creating the task should include all necessary context in the description, same as manually created tasks.

## Question #7: Malamar Agent Instruction Versioning

The Malamar agent instruction is hardcoded in the codebase. When Malamar is updated with new action types, is "update the binary, get new capabilities" the expected model?

### Answer

Yes, confirmed. The hardcoded instruction stays in sync with the binary. Remote knowledge base handles evolving best practices separately.

## Question #8: CLI Command Template Customization

Should users be able to customize CLI command templates, or are they intentionally locked down?

### Answer

Intentionally locked down for stability. CLI adapters are hardcoded. Users can customize binary paths and environment variables via CLI Settings, but not command templates.

## Question #9: SSE Event Broadcasting Scope

Should SSE events be scoped (e.g., only events for the current workspace), or is broadcasting everything acceptable?

### Answer

Keep broadcasting all events for now. Client-side filtering can handle noise if needed. Revisit scoping when adding multi-user authentication.

## Question #10: Chat Auto-Cleanup

Should there be auto-cleanup for old/inactive chats, or do they persist indefinitely?

### Answer

Chats persist indefinitely until manually deleted. No auto-cleanup needed given the lightweight nature of chat data.

## Question #11: Workspace With No Agents

What happens when a workspace has all its agents deleted?

### Answer

The system handles it gracefully:
- Chats can still happen with the Malamar agent (to recreate agents)
- Tasks would immediately move to "In Review" (no agents = all skip)
- UX improvement: Display a persistent, non-dismissable warning banner at the top of the workspace page when there are zero agents, until at least one agent is created.

## Question #12: Agent IDs in Task Input Files

Should task input files include agent IDs alongside names, like the chat context file does?

### Answer

No, keep names-only for task input files. Task agents communicate through comments (human-readable) and have no actions requiring IDs. IDs are only needed in chat context files where agents can perform structured actions.

## Question #13: Chat Error Notifications

Should chat errors trigger email notifications, or are notifications exclusively for task-related events?

### Answer

Notifications remain task-only. Chat is interactive and errors are immediately visible as system messages in the UI. Tasks are autonomous where the user might be away.

## Question #14: Mailgun Status Visibility

Should the Malamar agent have visibility into whether Mailgun is configured?

### Answer

Yes, confirmed. The chat context file includes Mailgun configuration status (configured/not configured), enabling the Malamar agent to warn users about missing setup without exposing credentials.

## Question #15: Chat Activity Logs

Should chats have a separate activity log table like tasks do?

### Answer

No, message history (including system messages) serves as the audit trail. Tasks need activity logs because comments are for collaboration, not system events. Chats blend both naturally in the message stream.

## Question #16: Chat Output File Naming

The chat output file uses `chat_id` while task output uses random nanoid. Should this be consistent?

### Answer

Yes, switching to random nanoid for chat output files: `/tmp/malamar_chat_output_{random_nanoid}.json`. This prevents the dangerous scenario where a failed CLI run leaves the previous output file intact, causing Malamar to parse stale data and potentially duplicate messages or actions.

Updated pattern:
- Chat input: `/tmp/malamar_chat_{chat_id}.md` (reused per chat)
- Chat output: `/tmp/malamar_chat_output_{random_nanoid}.json` (fresh each processing)

## Question #17: Temp File Cleanup

Should Malamar optionally clean up its own temp files, or is OS-dependent cleanup acceptable?

### Answer

OS-dependent `/tmp` cleanup is acceptable. Users needing control can use `MALAMAR_TEMP_DIR` to manage their own temp directory.

## Question #18: Inactive Chat State

Should there be an explicit "inactive" state for chats, or visual dimming for old chats?

### Answer

No explicit inactive state needed. Chats use implicit ordering (recently active at top). Old chats naturally sink and users can delete when no longer needed.

## Question #19: CLI Settings Refresh Button

Should there be a manual "Refresh CLI Status" button for immediate re-detection after installing a new CLI?

### Answer

Yes, add a "Refresh CLI Status" button on the CLI Settings page. This is low friction and gives users immediate feedback after installing new CLIs, while the 5-minute periodic check continues handling background changes.

## Question #20: Delete Workspace Action

Should the Malamar agent have a `delete_workspace` action?

### Answer

No, intentionally excluded. Workspace deletion is highly destructive (cascades everything - agents, tasks, comments, chats) and requires explicit human confirmation (typing workspace name). Giving agents the ability to delete workspaces would be catastrophic if hallucinated. The Malamar agent can create, modify, and organize, but destruction of top-level resources remains human-only.

## Question #21: Task Routing From Chat-Created Tasks

When a workspace agent creates a task via chat, should that task skip to the creating agent or start from the first agent?

### Answer

No special routing. All tasks follow the same loop starting from the first agent, regardless of origin (UI, Malamar agent action, or workspace agent action). Context goes in the description.
