# Session 007

## Agent Instructions Responsibility

Q: Looking at the agent instructions in SPECS.md, they describe high-level responsibilities like "enriching the task's summary" or "verifying feedback." But agents need to understand *when* to take action vs skip. How should agent instructions be written to help agents make good skip/comment/in_review decisions? Is there a pattern or guidance you'd want to provide to users setting up their agents?

A: It should - mostly - the user's responsibility in the agent's instruction to let them know WHEN to skip/comment/move the task to in review. The HOW question (responding in the correct format) is Malamar's responsibility.

---

## User Resources for Agent Instructions

Q: Given that the quality of agent decisions depends heavily on user-written instructions, do you envision Malamar providing any resources to help users write effective instructions? For example: documentation with examples, instruction templates beyond the default four agents, or a gallery of community-contributed agent configurations?

A: Alongside the repository and product, there will be a blog/documentation website describing detailed use case scenarios, best practices, etc. including resources on agent definitions. Within Malamar itself, two "aim far" items:

1. **AI-assisted agent definition**: A "chat-like" section in the Workspace Agents page where the user can chat with an available CLI. The CLI has access to resources explaining all aspects of Malamar, including best practices for writing comprehensive agent definitions. The CLI asks questions one by one to clarify and explore the user's needs, then generates a list of agents with well-crafted definitions.

2. **Community gallery of agent configurations**: A more complicated feature for the future, just noting it in the aim far section.

---

## Chat Feature Overview

Q: For the AI-assisted agent definition feature - when the user chats with the CLI to define agents, where would the conversation history live?

A: This expands into a broader "Chat" feature for workspaces. The workspace detail page will have a new "Chat" tab alongside "Tasks" and "Agents". Users can chat with:
- Any workspace agent
- A special "Malamar agent" designed to help manage the workspace and create/tune agents

---

## Chat Database Schema

Q: What tables are needed for the chat feature?

A: Two tables:

**chats**
- `id: string` (nanoid)
- `workspace_id: string` (foreign key)
- `agent_id?: string` (NULL if chatting with Malamar)
- `title: string` (default: "Untitled chat")
- `created_at: datetime`
- `updated_at: datetime`

**chat_messages**
- `id: string` (nanoid)
- `chat_id: string` (foreign key)
- `role: enum ("user" | "agent" | "system")`
- `message: string` (the conversational text)
- `actions: json?` (null for user/system, optional array for agent)
- `created_at: datetime`

---

## Chat Processing Mechanics

Q: How does chat processing work?

A: Similar to task processing:
1. Treated as a messaging model (not streaming) since CLIs are subprocesses
2. When user sends a message, Malamar collects all messages in the chat plus metadata (agent instruction, etc.)
3. Serialize into a markdown file at `/tmp/malamar_chat_{chat_id}.md`
4. Instruct the CLI to read from it and respond in a format Malamar can parse
5. Loop continues as conversation progresses

---

## Chat Actions

Q: Should the chat output format support actions similar to task agents?

A: Yes. The output format is:
```json
{
  "message": "I've created a Reviewer agent for TypeScript code review.",
  "actions": [
    { "type": "create_agent", "name": "Reviewer", "instruction": "...", "cli_type": "claude", "order": 3 }
  ]
}
```

The `message` field is encouraged but not required. Actions are displayed as structured cards/badges below the message text in the UI.

---

## Chat Action Types

Q: What action types are available for chat agents?

A:

**All chat agents (workspace agents + Malamar):**
- `rename_chat` - set the chat title
- `create_task` - create a new task in the workspace

**Malamar agent only:**
- `create_agent` - create a new agent in the workspace
- `update_agent` - modify an existing agent (instruction, name, cli_type)
- `delete_agent` - remove an agent
- `reorder_agents` - change agent execution order
- `update_workspace` - modify workspace settings

---

## Malamar Agent Context

Q: When chatting with the Malamar agent, how does it know the current workspace state?

A: A separate context file at `/tmp/malamar_chat_{chat_id}_context.md` contains:
- Workspace settings (title, description, working directory, cleanup settings, notifications)
- All agents with their instructions (ordered)
- Global settings (available CLIs and their health status, Mailgun configuration status)

The agent is instructed to read this file ONLY when needed. Task summaries are NOT included.

---

## Malamar Agent Knowledge Base

Q: How does the Malamar agent know best practices for creating agents, workspace setup, etc.?

A: Discovery-based remote knowledge base. The Malamar agent instruction includes a reference to a root URL (e.g., `https://github.com/malamar-dev/specs/blob/main/AGENTS/SKILLS/README.md`). This README links to other resources:
- Best practices for agents
- Workspace configuration guidance
- Troubleshooting
- Teaching users how to use Malamar

This allows the knowledge to stay up-to-date without updating Malamar itself.

---

## Chat Input File Format

Q: What format should the chat input file use?

A: Markdown for consistency with task input files:

```markdown
# Malamar Chat Context

You are being invoked by Malamar's chat feature.
{agent instruction here}

## Chat Metadata

- Chat ID: abc123
- Workspace: My Project
- Agent: Malamar (or agent name)

## Conversation History

​```json
{"role": "user", "content": "Help me create a reviewer agent", "created_at": "2025-01-17T10:00:00Z"}
{"role": "agent", "content": "Sure! What kind of work will this reviewer focus on?", "created_at": "2025-01-17T10:00:30Z"}
{"role": "user", "content": "Code review for TypeScript", "created_at": "2025-01-17T10:01:00Z"}
​```

## Additional Context (Malamar Agent Only)

For workspace state and settings, read: /tmp/malamar_chat_abc123_context.md

# Output Instruction

Write your response as JSON to: /tmp/malamar_chat_abc123_output.json
```

JSONL format for conversation history. System messages are excluded from the input file (agents don't need to see error history).

---

## Chat Queue

Q: How is chat processing tracked for cancellation?

A: A `chat_queue` table:

```
chat_queue
├── id: string (nanoid)
├── chat_id: string (foreign key)
├── workspace_id: string (foreign key)
├── status: enum ("queued" | "in_progress" | "completed" | "failed")
├── created_at: datetime
└── updated_at: datetime
```

No PID storage needed - the processor holds the subprocess reference in memory.

---

## Chat Concurrency

Q: How does chat concurrency work?

A:
- Chats within one workspace CAN process concurrently (unlike tasks which are one-at-a-time)
- No limit on concurrent chat processing across workspaces
- UI prevents sending multiple messages in the same chat while a response is pending

---

## Chat Error Handling

Q: How are errors displayed in chat?

A: System messages are inserted into the chat with `role: "system"` containing detailed error information (e.g., "Error: CLI exited with code 1" or "Error: Output file was empty"). These are visible to users in the chat UI but excluded from the input file when the conversation continues.

---

## Chat Title

Q: How are chats titled?

A: The agent can rename the chat via the `rename_chat` action. This is available to all chat agents (workspace agents + Malamar), allowing the agent to give the conversation a meaningful title based on context.

---

## Chat UI Navigation

Q: Looking at the workspace detail page, you mentioned there will be a "Chat" tab alongside "Tasks" and "Agents". When the user clicks on the Chat tab, what do they see? A list of previous chats, a way to start a new chat, or does it immediately open a new conversation?

A:
- When the user clicks on the Chat tab, displaying the list of previous chats. Ordering by recently active chat (chats that have recently added messages).
- Also have a dropdown button to create a new chat, the selectable items are Malamar Agent and all other workspace agents.
- Chat detail should be in a popup.

---

## Chat List Display

Q: For the chat list on the Chat tab - what information should be displayed for each chat item? Just the title, or also things like the agent name, last message preview, timestamp of last activity?

A: Title, agent name, last message/action preview, timestamp of last activity in human diff format.

---

## Chat Detail Popup Layout

Q: For the chat detail popup - what's the layout?

A: Header with chat title and agent name, message list in the middle (oldest on top for natural reading), and an input box at the bottom.

---

## Action Execution Confirmation

Q: When the Malamar agent returns actions like `create_agent`, `update_agent`, or `delete_agent` - are these executed immediately by Malamar, or should there be a confirmation step where the user approves the action before it takes effect?

A: Actions are executed immediately for now, for all agents, not just Malamar agent.

---

## Action Failure Handling

Q: If an action fails to execute (e.g., `create_agent` with a duplicate name, or `delete_agent` with an invalid agent ID), how should this be handled?

A: Adding a system message with the detailed error, similar to how task processing handles errors.

---

## Chat Deletion

Q: Can users delete chats? If so, what's the confirmation flow - simple confirm dialog, or typing the chat title like workspace deletion?

A: Yes, user can delete the entire chat. Simple confirm popup is good enough.

---

## Deleted Agent in Chat

Q: If a user has an ongoing chat with a workspace agent, and that agent gets deleted - what happens to the chat?

A:
1. Adding new feature to allow user to change the agent of the chat.
2. Meanwhile, when an agent is deleted for a chat, switch the chat back to the default agent: Malamar agent.

---

## Changing Chat Agent

Q: For the feature to change the chat's agent - where would this control live in the UI?

A: In the chat popup, the agent name should be displayed as a dropdown. The user can easily change the agent by just changing the dropdown value. Catch the onchange event and update the agent automatically.

---

## Conversation History After Agent Change

Q: When the user switches the chat's agent (via the dropdown), does the existing conversation history stay intact?

A: Only the agent is changed, there is nothing changed and no side-effects. The conversation history remains unchanged.

---

## Chat History Limits

Q: For the chat input file that's passed to the CLI - is there a limit on how many messages are included in the conversation history?

A: No limit, include all messages for now.

---

## Create Task Action Behavior

Q: When an agent uses the `create_task` action from chat, the task gets created in the workspace. Does the UI provide any feedback beyond the action card/badge in the chat?

A: Toast notification with a clickable link to open the newly created task. This requires a new SSE event type: `task.created` with the actor being the user. No side-effects.

---

## Renaming Chat via UI

Q: For the `rename_chat` action - can the user also manually rename a chat through the UI (not just via the agent action)?

A: Yes, changing inline by clicking the chat name. The UI converts the title to a text box with confirm button.

---

## Chat Context File for All Agents (Revision)

Q: The context file at `/tmp/malamar_chat_{chat_id}_context.md` was mentioned for the Malamar agent. Should other agents also have access to it?

A: Yes, the context file should be available to **all chat agents** (not just Malamar), since any agent can use `create_task` and may benefit from workspace context. The context file includes agent IDs alongside names, e.g., "Planner (id: 123456abcdef)".

---

## Chat Message Ordering

Q: In the chat detail popup, oldest messages on top. For the chat list page, when a new message is added to a chat, does that chat bubble up to the top of the list?

A: Yes, the chat with new messages will be bubbled up to the top of the list.

---

## Chat and Workspace Deletion

Q: When a workspace is deleted, what happens to its chats?

A: Cascade delete all chats and chat messages along with the workspace.

---

## Chat Queue Cleanup

Q: Should completed/failed chat queue items be cleaned up by the same daily Cleanup job that handles task queue items?

A: Yes, include into the Cleanup job. Delete completed/failed chat queue items older than 7 days.

---

## Canceling Chat Processing

Q: For tasks, there's a "Cancel" action to kill a running CLI subprocess. Should chats have a similar ability?

A: Yes:
- User is not allowed to send a new message if that chat has any queued/in_progress chat queue item.
- User can cancel the chat, which kills the CLI subprocess, changes the chat queue item to failed, then user can send another message to retrigger the agent.
- The "cancel" button should be a "stop" icon replacing the "send" icon in the message text area.

---

## Chat Processing Indicator

Q: While a chat is processing, should there be a visual indicator in the chat popup?

A: Just changing the send button to stop button, with some pulse animation is enough.

---

## File Attachment Feature

Q: Is there a file attachment feature for chat?

A: Yes. The user is able to upload a file into the chat (any file format). The uploading directly places the file into `/tmp/malamar_chat_{chat_id}_attachments/{filename}`, then creates a system message noting: "User has uploaded /tmp/malamar_chat_{chat_id}_attachments/{filename}". The user and agent can then continue with the file mentioned any time.

---

## File Attachment - Multiple Files

Q: Can users upload multiple files? If so, does each file create its own system message?

A: Yes, each file creates its own system message.

---

## System Messages in Input File (Revision)

Q: Previously it was stated that system messages are excluded from the chat input file. With file attachments, system messages contain important context. Should this be revised?

A: Yes, **include all system messages** in the chat input file. This is a revision from the earlier decision.

---

## File Attachment - Duplicate Filenames

Q: If a user uploads a file with the same filename as a previously uploaded file in the same chat, what happens?

A: Overwrite the existing file, as there is no delete attachment functionality for now.

---

## File Attachment - UI Placement

Q: Where in the chat UI should the file upload control be?

A: Attachment icon next to the message input box, similar to most chat applications.

---

## File Attachment - Size Limits

Q: Should there be a file size limit for attachments?

A: No limit for now, keep it simple.

---

## File Attachment Cleanup

Q: When a chat is deleted, should the attachment directory be deleted?

A: Just let the OS handle `/tmp` cleanup naturally, consistent with other temp files.

---

## Chat SSE Events

Q: Should there be SSE events for chat activity?

A: Yes, adding these new SSE events:
- `chat.message_added` - when agent responds or user/system adds message
- `chat.processing_started` - when CLI starts processing
- `chat.processing_finished` - when CLI finishes

---

## Chat SSE Event Payloads

Q: What should the payloads for chat SSE events look like?

A:
- `chat.message_added`: `chat_id`, `chat_title`, `author_type` (user/agent/system), `workspace_id`
- `chat.processing_started`: `chat_id`, `chat_title`, `agent_name`, `workspace_id`
- `chat.processing_finished`: `chat_id`, `chat_title`, `agent_name`, `workspace_id`

---

## Action Target Identification

Q: For Malamar agent actions like `update_agent` or `delete_agent`, should the agent specify the target by `agent_id` or by `agent_name`?

A: By `agent_id`. The context file includes agent IDs alongside names, e.g., "Planner (id: 123456abcdef)".

---

## Action Schemas - Agent Actions

Q: What are the schemas for agent-related actions?

A:
- `create_agent`: `{ "type": "create_agent", "name": "...", "instruction": "...", "cli_type": "claude|gemini|codex|opencode", "order": number }`
- `update_agent`: `{ "type": "update_agent", "agent_id": "...", "name?": "...", "instruction?": "...", "cli_type?": "..." }`
- `delete_agent`: `{ "type": "delete_agent", "agent_id": "..." }`
- `reorder_agents`: `{ "type": "reorder_agents", "agent_ids": ["id1", "id2", "id3"] }` (array in desired order)

---

## Action Schemas - Other Actions

Q: What are the schemas for the remaining actions?

A:
- `rename_chat`: `{ "type": "rename_chat", "title": "..." }`
- `create_task`: `{ "type": "create_task", "summary": "...", "description": "..." }`
- `update_workspace`: `{ "type": "update_workspace", "title?": "...", "description?": "...", "working_directory_mode?": "static|temp", "working_directory_path?": "...", ... }` (allows updating all workspace settings including cleanup and notification settings)

---

## Malamar Agent CLI Selection

Q: What CLI does the Malamar agent use?

A:
1. Malamar agent always uses, in order, the first available CLI: Claude Code → Gemini CLI → Codex CLI → OpenCode.
2. In the chat popup, next to the agent selector, there is also a CLI selector to override which CLI handles the chat:
   - When changing the agent, automatically update the CLI corresponding to that agent's configured CLI.
   - When manually changing the CLI, save it to the chat as an override.

---

## Chat Schema (Revised)

Q: What is the updated chat schema reflecting the CLI selection?

A:
```
chats
├── id: string (nanoid)
├── workspace_id: string (foreign key)
├── agent_id?: string (NULL if chatting with Malamar)
├── cli_type?: string (NULL = respect agent's CLI or Malamar default)
├── title: string (default: "Untitled chat")
├── created_at: datetime
└── updated_at: datetime
```

**Behavior:**
1. **Malamar agent** (agent_id is NULL): Uses first available CLI in order: Claude Code → Gemini CLI → Codex CLI → OpenCode
2. **Workspace agents**: Uses the CLI configured on the agent
3. **CLI override** (cli_type is not NULL): Overrides both of the above, uses the specified CLI regardless of agent configuration

**UI behavior:**
- Agent dropdown and CLI dropdown side by side
- Changing agent → auto-updates CLI dropdown to match that agent's configured CLI (sets cli_type to NULL)
- Manually changing CLI → saves cli_type to the chat

---

## Malamar Agent Instruction

Q: Where does the Malamar agent's instruction come from?

A: Hardcoded in Malamar's codebase, with extra instructions pointing to the remote knowledge base URL for discovering best practices and documentation.

---

## Chat in Workspace Without Agents

Q: If a workspace has all its agents deleted, can the user still create a new chat?

A: Yes. The user can still chat with Malamar agent to create new agents.

---

## Chat Working Directory

Q: When chatting with an agent, what working directory does the CLI use?

A: The chat respects the workspace's working directory setting:

1. **Temp Folder mode**: Creates `/tmp/malamar_chat_{chat_id}` as the working directory for that chat
2. **Static Directory mode**: Uses the directory defined in the workspace settings

This enables the ability to talk to workspace agents with full context and working setup for that workspace. For example, if the workspace points to a repository, the agent can browse files, run commands, and make changes in that repository during the chat.
