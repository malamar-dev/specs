# Session 008

## Chat Processing Timeout

Q: For chat processing, is there any timeout consideration, or is it purely user-initiated cancellation like tasks?

A: No timeout consideration for chat processing, purely user-initiated cancellation via the stop button.

---

## Remote Knowledge Base Failures

Q: When the Malamar agent fetches the remote knowledge base URL and the network is unavailable or the URL returns an error, should this be treated as a silent failure or surface as a system message?

A: Should be treated as a silent failure. The Malamar agent proceeds without the knowledge base if unreachable.

---

## Reorder Agents Validation

Q: For the `reorder_agents` action, what happens if the array is incomplete (missing some agent IDs) or contains invalid IDs?

A: Reject the entire action with a system message. The action must include all valid agent IDs in the workspace.

---

## Mid-Flight Working Directory Changes

Q: If the working directory is changed (via Malamar agent action or manual UI) while a task is actively being processed, what happens?

A: Accept the risk. The currently running agent continues with the old directory; subsequent agents pick up the new path. This applies to both Malamar agent actions and manual user changes.

---

## Agent Switch System Message

Q: When a user switches agents mid-conversation in a chat, should the input file indicate the context shift?

A: Yes, adding a new system message like "Changed agent from Planner to Malamar" when the user switches agents. Important: Only user messages trigger CLI responses - system messages NEVER trigger the agent to respond.

---

## Create Task Immediate Processing

Q: When an agent uses the `create_task` action from chat, should there be a delay or draft state before runner pickup?

A: No, immediate processing is expected. The agent creating the task should include all necessary context in the description, same as manually created tasks.

---

## Malamar Agent Instruction Versioning

Q: The Malamar agent instruction is hardcoded in the codebase. When Malamar is updated with new action types, is "update the binary, get new capabilities" the expected model?

A: Yes, confirmed. The hardcoded instruction stays in sync with the binary. Remote knowledge base handles evolving best practices separately.

---

## CLI Command Template Customization

Q: Should users be able to customize CLI command templates, or are they intentionally locked down?

A: Intentionally locked down for stability. CLI adapters are hardcoded. Users can customize binary paths and environment variables via CLI Settings, but not command templates.

---

## SSE Event Broadcasting Scope

Q: Should SSE events be scoped (e.g., only events for the current workspace), or is broadcasting everything acceptable?

A: Keep broadcasting all events for now. Client-side filtering can handle noise if needed. Revisit scoping when adding multi-user authentication.

---

## Chat Auto-Cleanup

Q: Should there be auto-cleanup for old/inactive chats, or do they persist indefinitely?

A: Chats persist indefinitely until manually deleted. No auto-cleanup needed given the lightweight nature of chat data.

---

## Workspace With No Agents

Q: What happens when a workspace has all its agents deleted?

A: The system handles it gracefully:
- Chats can still happen with the Malamar agent (to recreate agents)
- Tasks would immediately move to "In Review" (no agents = all skip)
- UX improvement: Display a persistent, non-dismissable warning banner at the top of the workspace page when there are zero agents, until at least one agent is created.

---

## Agent IDs in Task Input Files

Q: Should task input files include agent IDs alongside names, like the chat context file does?

A: No, keep names-only for task input files. Task agents communicate through comments (human-readable) and have no actions requiring IDs. IDs are only needed in chat context files where agents can perform structured actions.

---

## Chat Error Notifications

Q: Should chat errors trigger email notifications, or are notifications exclusively for task-related events?

A: Notifications remain task-only. Chat is interactive and errors are immediately visible as system messages in the UI. Tasks are autonomous where the user might be away.

---

## Mailgun Status Visibility

Q: Should the Malamar agent have visibility into whether Mailgun is configured?

A: Yes, confirmed. The chat context file includes Mailgun configuration status (configured/not configured), enabling the Malamar agent to warn users about missing setup without exposing credentials.

---

## Chat Activity Logs

Q: Should chats have a separate activity log table like tasks do?

A: No, message history (including system messages) serves as the audit trail. Tasks need activity logs because comments are for collaboration, not system events. Chats blend both naturally in the message stream.

---

## Chat Output File Naming

Q: The chat output file uses `chat_id` while task output uses random nanoid. Should this be consistent?

A: Yes, switching to random nanoid for chat output files: `/tmp/malamar_chat_output_{random_nanoid}.json`. This prevents the dangerous scenario where a failed CLI run leaves the previous output file intact, causing Malamar to parse stale data and potentially duplicate messages or actions.

Updated pattern:
- Chat input: `/tmp/malamar_chat_{chat_id}.md` (reused per chat)
- Chat output: `/tmp/malamar_chat_output_{random_nanoid}.json` (fresh each processing)

---

## Temp File Cleanup

Q: Should Malamar optionally clean up its own temp files, or is OS-dependent cleanup acceptable?

A: OS-dependent `/tmp` cleanup is acceptable. Users needing control can use `MALAMAR_TEMP_DIR` to manage their own temp directory.

---

## Inactive Chat State

Q: Should there be an explicit "inactive" state for chats, or visual dimming for old chats?

A: No explicit inactive state needed. Chats use implicit ordering (recently active at top). Old chats naturally sink and users can delete when no longer needed.

---

## CLI Settings Refresh Button

Q: Should there be a manual "Refresh CLI Status" button for immediate re-detection after installing a new CLI?

A: Yes, adding a "Refresh CLI Status" button on the CLI Settings page. This is low friction and gives users immediate feedback after installing new CLIs, while the 5-minute periodic check continues handling background changes.

---

## Delete Workspace Action

Q: Should the Malamar agent have a `delete_workspace` action?

A: No, intentionally excluded. Workspace deletion is highly destructive (cascades everything - agents, tasks, comments, chats) and requires explicit human confirmation (typing workspace name). Giving agents the ability to delete workspaces would be catastrophic if hallucinated. The Malamar agent can create, modify, and organize, but destruction of top-level resources remains human-only.

---

## Task Routing From Chat-Created Tasks

Q: When a workspace agent creates a task via chat, should that task skip to the creating agent or start from the first agent?

A: No special routing. All tasks follow the same loop starting from the first agent, regardless of origin (UI, Malamar agent action, or workspace agent action). Context goes in the description.
