# Session 004

Please read [SKILL.md](../.claude/skills/explore-specs/SKILL.md) to understand the process used to create this file.

## Question #1: Agent Modification During Active Processing

When an agent is deleted or modified while a task is actively being processed by the runner, what should happen?

### Answer

The runner uses a just-in-time snapshot approach:

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

## Question #2: Next Agent Lookup

For the "next agent snapshot" lookup - how does the runner determine which agent is next?

### Answer

Query for "the agent with the smallest `order` value that is greater than the current agent's `order`". This is fully dynamic - if a new agent is inserted mid-loop with an `order` value between the current and next agent, it will be picked up.

## Question #3: Other Agents Brief Purpose

What is the purpose of including "a brief of other agents in the same workspace (names only)" in the input file?

### Answer

Two purposes:
1. **Workflow awareness** - The agent knows who else exists in the workflow
2. **Addressing comments** - The agent can direct comments to specific agents (e.g., "Reviewer, please check X")

The list reflects the agents at the moment of input file creation.

## Question #4: Input File Markdown Structure

What should the input file markdown structure look like?

### Answer

The structure is:

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

Comments are in JSONL format (one JSON object per line) inside a code block to prevent markdown content from breaking the file structure.

## Question #5: Comment Author Field Values

For the `author` field in comments - what values should be used?

### Answer

- **Agent comments**: The agent's name at the time the comment was created
- **System comments**: "System"
- **User comments**: "User"

## Question #6: Malformed Output Handling

When the runner reads the output file and the JSON is malformed (empty, truncated, wrong structure, missing file) - what happens?

### Answer

The error handling is consistent with CLI crashes/timeouts:
1. Write system comment with detailed error message
2. Stop the current loop immediately
3. The comment emits a task event → new queue item
4. New loop is triggered via that queue item

## Question #7: Temp Folder Cleanup

When should temp folders (`/tmp/malamar_tasks_{task_id}`) be deleted?

### Answer

Never. Let the OS handle `/tmp` cleanup naturally.

## Question #8: Kill Loop Mechanics

How does Malamar terminate a running CLI subprocess when the user clicks "Kill Loop"?

### Answer

Send SIGTERM to the process. Leave the output file as-is. No forced SIGKILL, no cleanup.

## Question #9: CLI Detection Mechanism

How does Malamar determine which CLIs are available?

### Answer

Two-layer detection:

1. **Auto-detection**: Search PATH for the binary, run a test prompt (e.g., "Respond with OK and exit"), validate response, store version in memory
2. **Manual settings**: Per-CLI configuration page for custom binary path and environment variables

Binary names:
- Claude Code: `claude`
- Gemini CLI: `gemini`
- Codex CLI: `codex`
- OpenCode: `opencode`

## Question #10: CLI Settings Fields

What fields should each CLI's settings section have?

### Answer

- **Binary Path**: Custom path override (empty = use PATH)
- **Environment Variables**: Key-value pairs to inject when running
- **Detected Version**: Read-only, shows auto-detected version
- **Status**: Read-only ("Available", "Not Found", "Test Failed")

No "Enabled" toggle - if it passes the test, it's available for agent assignment.

## Question #11: Unavailable CLI Warning

Should the UI warn when an assigned CLI becomes unavailable?

### Answer

Yes, warnings should appear on:
- Workspace detail page
- Create task form
- Task detail page

## Question #12: Default Agent Instructions

Do default agents (Planner, Implementer, Reviewer, Approver) come with default instructions?

### Answer

Yes, they come with the default instructions documented in SPECS.md. These will be polished later.

## Question #13: SSE Event Types

What events should the backend emit via SSE?

### Answer

Events with payloads for toast display:
- `task.status_changed`: task_id, task_summary, old_status, new_status, workspace_id
- `task.comment_added`: task_id, task_summary, author_name, workspace_id (no comment content)
- `task.error_occurred`: task_id, task_summary, error_message, workspace_id
- `agent.execution_started`: task_id, task_summary, agent_name, workspace_id
- `agent.execution_finished`: task_id, task_summary, agent_name, workspace_id

## Question #14: Workspace Deletion Cascade

When a workspace is deleted, what happens to its tasks, agents, and queue items?

### Answer

Force kill, then cascade delete:
1. User must type the workspace name to confirm
2. Force kill any running CLI subprocess (SIGTERM)
3. Cascade delete: agents, tasks, comments, queue items - everything

## Question #15: Delete In-Progress Task

Can a user delete a task while it's "In Progress"?

### Answer

Yes, same pattern - force kill, confirm, then delete (task + comments + queue items).

## Question #16: Queue Item States

What states can a queue item be in?

### Answer

Four states:
- `queued`: Waiting to be picked up
- `in_progress`: Currently being processed
- `completed`: Finished successfully
- `failed`: Processing failed

Completed/failed items are kept to support the "prioritize most recently processed task" logic.

## Question #17: Queue Cleanup

How does queue cleanup work?

### Answer

A separate background job on a cron schedule (e.g., daily) deletes `completed` and `failed` queue items older than 7 days. `queued` and `in_progress` items are left untouched.

## Question #18: Background Jobs

What background jobs does Malamar run?

### Answer

1. **Runner**: Picks up queue items, executes agent loops (continuous loop with 1s sleep)
2. **Queue cleanup**: Cron job to delete old completed/failed queue items
3. **CLI health check**: Runs every 5 minutes, updates in-memory state

Email notifications are fire-and-forget async calls, not a separate job.

## Question #19: CLI Health Check and UI

How does CLI health check work with the UI?

### Answer

CLI health check updates in-memory state. UI polls a health endpoint to get current CLI status. No SSE push for CLI changes.

## Question #20: Runner Workspace Concurrency

How does the runner handle multiple workspaces?

### Answer

Single runner process that spawns concurrent workers per workspace. Each workspace can have one task being processed concurrently with other workspaces. No artificial limit on concurrent workspace workers.

## Question #21: Task Status Backwards Transition

Can a task move backwards from "Done"?

### Answer

Yes, user can move a task from "Done" to "In Progress" or "Todo" after commenting or editing. No restrictions.

Additional notes:
1. When runner picks a task for a new loop, all other "In Progress" tasks in that workspace are moved to "Todo" automatically
2. User can "prioritize" a task for the next loop via a button, overriding normal pickup behavior

## Question #22: Priority Override Mechanics

How does the priority override work mechanically?

### Answer

The `is_priority` flag lives on the queue item:
1. User clicks "prioritize" on a task
2. Find or create queue item for that task
3. Set `is_priority: true` on that queue item
4. Set `is_priority: false` on all other queue items in that workspace
5. Runner checks for `is_priority: true` first when picking

Only one prioritized task per workspace at a time. After processing, returns to normal pickup behavior.

## Question #23: Full Queue Pickup Priority

What is the full priority order for queue pickup?

### Answer

Per workspace, in order:
1. Any queue item with `is_priority: true` and status `queued`
2. Task that most recently has a `completed` or `failed` queue item → pick its `queued` item
3. Most recently updated `queued` item (LIFO by `updated_at`)

## Question #24: Duplicate Queue Item Prevention

When a task event is emitted and there's already a `queued` item for that task, what happens?

### Answer

Don't create a new queue item, but update the `updated_at` of the existing `queued` item. This bumps it up in the LIFO priority order.

## Question #25: Mailgun Settings

What Mailgun settings does the user need to provide?

### Answer

- API key
- Domain
- From email address
- To email address (recipient)
- "Send test email" button to verify configuration

## Question #26: Per-Workspace Notification Overrides

How do per-workspace notification overrides work?

### Answer

Workspace can only toggle events on/off (inherits global email config). Settings per workspace:
- On error occurred: on/off (default: inherit global)
- On task moved to in review: on/off (default: inherit global)

## Question #27: Deleted Agent Comment Display

When displaying a comment made by a deleted agent, what should happen?

### Answer

Show "(Deleted Agent)" as the author name. The comment keeps the `agent_id` for reference, but display gracefully handles missing agents.

## Question #28: Live Editing During Loop

Can workspace/task description be edited while a loop is running?

### Answer

Yes, and the new data will be reflected to the next agent in the loop (due to just-in-time snapshot approach).

## Question #29: Field Length Limits

Are there maximum length limits for fields?

### Answer

No limits in the application layer. All fields (task description, comments, agent instructions) support long content.

## Question #30: Hanging CLI Process

What happens if the CLI process hangs and never exits?

### Answer

Nothing special - respect the current design. The process runs until it exits or the user manually kills the loop.
