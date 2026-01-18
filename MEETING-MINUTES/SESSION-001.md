# Session 001

Please read [SKILL.md](../.claude/skills/explore-specs/SKILL.md) to understand the process used to create this file.

## Question #1: Primary User Flow to Optimize

What is the primary user experience that Malamar aims to optimize?

### Answer

The primary user flow to optimize is running the multi-agent loop automatically, with the runner orchestrating agents end-to-end without manual intervention.

## Question #2: Loop Termination Condition

What ends a loop iteration?

### Answer

The loop ends when the task is moved to "In Review" by an agent or runner action. Only the human is allowed to mark a task "Done" - agents cannot transition a task directly to the Done status.

## Question #3: "Nothing to Do" Signal

What is the "nothing to do" signal that indicates the loop should stop?

### Answer

The "nothing to do" signal occurs when all agents choose "skip" in their response and no new comment is added to the task. This indicates that no agent has meaningful work to contribute, and the loop should terminate.

## Question #4: Loop Continuation After Agent Comment

If an agent adds a comment, how does the loop proceed?

### Answer

When an agent adds a comment, the loop continues to the next agent immediately, passing the updated task data (including the new comment) to that next agent. After all agents finish the current pass, the runner retriggers the loop again with the updated task state. This ensures each agent sees the latest context from previous agents in the same pass.

## Question #5: Loop Restart Position

When the loop is retriggered, where does it start?

### Answer

When retriggered, the loop always starts from the first agent in the workspace order. There is no resumption from where it left off - each new loop pass begins at agent #1.

## Question #6: Agent Execution Model

Are agents executed sequentially or in parallel?

### Answer

Agents are executed sequentially, one agent at a time. This ensures each agent can see the results of previous agents before making its own decisions.

## Question #7: Iteration Limits

Is there a hard cap on loop iterations?

### Answer

No, there is no hard cap on iteration count or time. The loop continues until it reaches a termination condition (task moved to "In Review" or all agents skip).

## Question #8: New Task Events During Active Loop

What happens if a new task event arrives during an active loop?

### Answer

The runner finishes the current loop, then restarts from agent #1. This follows the task event queue rules - new events are queued and processed after the current loop completes.

## Question #9: Agent Status Change Capability

Can agents change task status?

### Answer

Yes, but only by requesting a move to "In Review" via the agent response format. No other status changes are allowed from agents - they cannot move tasks to "Todo", "In Progress", or "Done".

## Question #10: Comment with "In Review" Request

If an agent comments and requests "In Review," what happens?

### Answer

The runner stops the current loop immediately. Even if the runner already asserted task properties before calling the agent, it stops right away. The next queue item will be picked up, but no agents will run on that task because it is already in "In Review" status.

## Question #11: Agents on "In Review" Tasks

Do agents work on tasks already in "In Review"?

### Answer

No, agents are not allowed to run on tasks in "In Review" status. These tasks are waiting for human attention and are excluded from agent processing.

## Question #12: Agent Failure and Timeout Handling

How are agent failures and timeouts handled?

### Answer

When an agent fails or times out, the runner:
1. Writes a System comment with detailed error/timeout information
2. Emits a new task event into the queue
3. Stops the current loop

A new loop will be triggered through the queue (contextual retry forever), without moving the task to "In Review". This allows the system to recover from transient failures.

## Question #13: Retry Backoff

Is there backoff for retries?

### Answer

No explicit retry or backoff mechanism. The task event queue provides natural delay between retry attempts.

## Question #14: System Comment Format

Should system comments be structured or free text?

### Answer

Free text only. System comments are human-readable messages that describe what happened (errors, status changes, etc.).

## Question #15: Comment Fields

What fields should a comment have?

### Answer

Comments have the following fields:
- `task_id: string` - The task this comment belongs to
- `workspace_id: string` - The workspace this comment belongs to
- `user_id?: string` - NULL if system or agent; if present, always `000000000000000000000` (21 zeros, a mock nanoid)
- `agent_id?: string` - NULL if system or user; contains the agent's nanoid if from an agent
- `content: string` - Long markdown string
- `created_at: Date`
- `updated_at: Date`

## Question #16: Interpreting user_id and agent_id

How should `user_id` and `agent_id` be interpreted?

### Answer

- `user_id` is NULL for system and agent comments; if present, it is always the mock nanoid of all zeros (representing the single user)
- `agent_id` is NULL for system and user comments; if present, it contains the agent's nanoid
- `content` is long markdown text that can contain any formatting

## Question #17: Multiple Users

Will there be multiple users later?

### Answer

Yes, multiple users are planned for the future. For now, there is only one user with no authentication, so the single mock user ID (`000000000000000000000`) is used everywhere.

## Question #18: Mock user_id Configuration

Should the mock `user_id` be a config value?

### Answer

Hardcoded is acceptable for now. The mock user ID can remain a constant in the codebase until multi-user support is implemented.

## Question #19: Agent Identification

How are agents identified?

### Answer

Each agent has its own nanoid. The default example agents (Planner, Implementer, Reviewer, Approver) are auto-created per workspace, but users can edit, delete, add, and reorder them as needed.

## Question #20: Agent Order Field

Is an explicit agent `order` field required?

### Answer

Yes, an explicit `order` field is required and must be unique within the workspace. When reordering agents, the client/UI must send all the new valid ordering to ensure consistency.

## Question #21: Agent Response Format

What is the agent response format?

### Answer

Agents respond with a JSON structure containing an array of actions. This allows multiple actions in a single response (e.g., comment AND request "In Review").

**Structure:**
```json
{
  "actions": [
    { "type": "skip" },
    { "type": "comment", "content": "<markdown string>" },
    { "type": "change_status", "status": "in_review" }
  ]
}
```

## Question #22: Available Agent Action Types

What action types are available for agents?

### Answer

Three action types are available:
- `skip` - Agent has nothing to do
- `comment` - Agent did meaningful work and has progress to report
- `change_status` - Agent requests to move the task (only "in_review" is allowed as the status value)

## Question #23: Valid Action Combinations

What are the valid action combinations?

### Answer

Four valid combinations:
1. `skip` alone - Agent has nothing to do
2. `comment` alone - Agent did work, loop continues to next agent
3. `comment` + `change_status: "in_review"` - Agent comments and requests human attention
4. `change_status: "in_review"` alone - Technically valid but discouraged; agents should always explain why human attention is needed

## Question #24: "Nothing to Do" Comments

Should agents comment "I have nothing to do"?

### Answer

No. If an agent has nothing to do, it should use the `skip` action. Agents should never comment "I have nothing to do" or similar messages - the skip action is the proper way to signal inactivity.