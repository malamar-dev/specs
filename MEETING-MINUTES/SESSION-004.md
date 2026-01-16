# Session 004

## Agent and Loop Dynamics

Q: When an agent is deleted or modified while a task is actively being processed by the runner, what should happen?

A: The runner uses a just-in-time snapshot approach:
1. A new loop starts
2. Runner gets the first agent snapshot
3. Runner gets the current data of the task
4. Runner executes the agent with the task data
5. Runner performs the actions from the agent result
6. Runner gets the next agent snapshot (if none exists, skip)
7. Runner gets the current task data (including any new comments from step 5)
8. Runner executes the agent with the task data
9. Loop until no more agents
10. If all agents returned skip action, move task to "In Review"
11. Exit for new loop

If an agent is deleted mid-loop, it's simply not found at step 6 and skipped. Changes to an agent's instruction don't affect a currently running execution since the snapshot was already taken.

---

Q: For the "next agent snapshot" lookup - how does the runner determine which agent is next?

A: Query for "the agent with the smallest `order` value that is greater than the current agent's `order`". This is fully dynamic - if a new agent is inserted mid-loop with an `order` value between the current and next agent, it will be picked up.

---

## Input File Structure

Q: What is the purpose of including "a brief of other agents in the same workspace (names only)" in the input file?

A: Two purposes:
1. Workflow awareness - agent knows who else exists in the workflow
2. Addressing comments - agent can direct comments to specific agents (e.g., "Reviewer, please check X")

The list reflects the agents at the moment of input file creation.

---

Q: What should the input file markdown structure look like?

A: The structure is:

```markdown
# Malamar Context
You are being orchestrated by Malamar...
{workspace instruction here}

# Your Role
{agent instruction here}

## Other Agents in This Workflow
- Planner
- Reviewer
- Approver

# Task
## Summary
{task summary}

## Description
{task description}

## Comments

​```json
{"author": "Planner", "agent_id": "abc123", "content": "## My Plan\n\n1. First step", "created_at": "2025-01-17T10:00:00Z"}
{"author": "User", "user_id": "000000000000000000000", "content": "Looks good!", "created_at": "2025-01-17T10:05:00Z"}
{"author": "System", "content": "Error: CLI timeout", "created_at": "2025-01-17T10:10:00Z"}
​```

# Output Instruction
Write your response to: /tmp/malamar_output_xxx.json
```

Comments are JSONL format (one JSON object per line) inside a code block to prevent markdown content from breaking the file structure.

---

Q: For the `author` field in comments - what values should be used?

A:
- Agent comments: the agent's name at the time the comment was created
- System comments: "System"
- User comments: "User"

---

## Error Handling

Q: When the runner reads the output file and the JSON is malformed (empty, truncated, wrong structure, missing file) - what happens?

A: The error handling is consistent with CLI crashes/timeouts:
1. Write system comment with detailed error message
2. Stop the current loop immediately
3. The comment emits a task event → new queue item
4. New loop is triggered via that queue item

---

## Temp Folder Cleanup

Q: When should temp folders (`/tmp/malamar_tasks_{task_id}`) be deleted?

A: Never. Let the OS handle `/tmp` cleanup naturally.

---

## Kill Loop Feature

Q: How does Malamar terminate a running CLI subprocess when the user clicks "Kill Loop"?

A: Send SIGTERM to the process. Leave the output file as-is. No forced SIGKILL, no cleanup.

---

## CLI Detection and Settings

Q: How does Malamar determine which CLIs are available?

A: Two-layer detection:
1. **Auto-detection**: Search PATH for the binary, run a test prompt (e.g., "Respond with OK and exit"), validate response, store version in memory
2. **Manual settings**: Per-CLI configuration page for custom binary path and environment variables

Binary names:
- Claude Code: `claude`
- Gemini CLI: `gemini`
- Codex CLI: `codex`
- OpenCode: `opencode`

---

Q: What fields should each CLI's settings section have?

A:
- Binary Path: custom path override (empty = use PATH)
- Environment Variables: key-value pairs to inject when running
- Detected Version: read-only, shows auto-detected version
- Status: read-only ("Available", "Not Found", "Test Failed")

No "Enabled" toggle - if it passes the test, it's available for agent assignment.

---

Q: Should the UI warn when an assigned CLI becomes unavailable?

A: Yes, warnings should appear on:
- Workspace detail page
- Create task form
- Task detail page

---

## Default Agents

Q: Do default agents (Planner, Implementer, Reviewer, Approver) come with default instructions?

A: Yes, they come with the default instructions documented in SPECS.md. These will be polished later.

---

## SSE Events

Q: What events should the backend emit via SSE?

A: Events with payloads for toast display:
- `task.status_changed`: task_id, task_summary, old_status, new_status, workspace_id
- `task.comment_added`: task_id, task_summary, author_name, workspace_id (no comment content)
- `task.error_occurred`: task_id, task_summary, error_message, workspace_id
- `agent.execution_started`: task_id, task_summary, agent_name, workspace_id
- `agent.execution_finished`: task_id, task_summary, agent_name, workspace_id

---

## Deletion Behavior

Q: When a workspace is deleted, what happens to its tasks, agents, and queue items?

A: Force kill, then cascade delete:
1. User must type the workspace name to confirm
2. Force kill any running CLI subprocess (SIGTERM)
3. Cascade delete: agents, tasks, comments, queue items - everything

---

Q: Can a user delete a task while it's "In Progress"?

A: Yes, same pattern - force kill, confirm, then delete (task + comments + queue items).

---

## Task Event Queue

Q: What states can a queue item be in?

A: Four states:
- `queued`: waiting to be picked up
- `in_progress`: currently being processed
- `completed`: finished successfully
- `failed`: processing failed

Completed/failed items are kept to support the "prioritize most recently processed task" logic.

---

Q: How does queue cleanup work?

A: A separate background job on a cron schedule (e.g., daily) deletes `completed` and `failed` queue items older than 7 days. `queued` and `in_progress` items are left untouched.

---

## Background Jobs

Q: What background jobs does Malamar run?

A:
1. **Runner**: picks up queue items, executes agent loops (continuous loop with 1s sleep)
2. **Queue cleanup**: cron job to delete old completed/failed queue items
3. **CLI health check**: runs every 5 minutes, updates in-memory state

Email notifications are fire-and-forget async calls, not a separate job.

---

Q: How does CLI health check work with the UI?

A: CLI health check updates in-memory state. UI polls a health endpoint to get current CLI status. No SSE push for CLI changes.

---

## Concurrent Processing

Q: How does the runner handle multiple workspaces?

A: Single runner process that spawns concurrent workers per workspace. Each workspace can have one task being processed concurrently with other workspaces. No artificial limit on concurrent workspace workers.

---

## Task Status Transitions

Q: Can a task move backwards from "Done"?

A: Yes, user can move a task from "Done" to "In Progress" or "Todo" after commenting or editing. No restrictions.

Additional notes:
1. When runner picks a task for a new loop, all other "In Progress" tasks in that workspace are moved to "Todo" automatically
2. User can "prioritize" a task for the next loop via a button, overriding normal pickup behavior

---

## Priority Override

Q: How does the priority override work mechanically?

A: The `is_priority` flag lives on the queue item:
1. User clicks "prioritize" on a task
2. Find or create queue item for that task
3. Set `is_priority: true` on that queue item
4. Set `is_priority: false` on all other queue items in that workspace
5. Runner checks for `is_priority: true` first when picking

Only one prioritized task per workspace at a time. After processing, returns to normal pickup behavior.

---

## Queue Pickup Priority

Q: What is the full priority order for queue pickup?

A: Per workspace, in order:
1. Any queue item with `is_priority: true` and status `queued`
2. Task that most recently has a `completed` or `failed` queue item → pick its `queued` item
3. Most recently updated `queued` item (LIFO by `updated_at`)

---

Q: When a task event is emitted and there's already a `queued` item for that task, what happens?

A: Don't create a new queue item, but update the `updated_at` of the existing `queued` item. This bumps it up in the LIFO priority order.

---

## Mailgun Configuration

Q: What Mailgun settings does the user need to provide?

A:
- API key
- Domain
- From email address
- To email address (recipient)
- "Send test email" button to verify configuration

---

Q: How do per-workspace notification overrides work?

A: Workspace can only toggle events on/off (inherits global email config). Settings per workspace:
- On error occurred: on/off (default: inherit global)
- On task moved to in review: on/off (default: inherit global)

---

## Deleted Agent Handling

Q: When displaying a comment made by a deleted agent, what should happen?

A: Show "(Deleted Agent)" as the author name. The comment keeps the `agent_id` for reference, but display gracefully handles missing agents.

---

## Miscellaneous

Q: Can workspace/task description be edited while a loop is running?

A: Yes, and the new data will be reflected to the next agent in the loop (due to just-in-time snapshot approach).

---

Q: Are there maximum length limits for fields?

A: No limits in the application layer. All fields (task description, comments, agent instructions) support long content.

---

Q: What happens if the CLI process hangs and never exits?

A: Nothing special - respect the current design. The process runs until it exits or the user manually kills the loop.
