# Session 001

## Loop Behavior

Q: What is the primary user flow to optimize?

A: Running the multi-agent loop automatically, with the runner orchestrating agents end-to-end.

---

Q: What ends a loop iteration?

A: The loop ends when the task is moved to "In Review" by an agent/runner action. Only the human is allowed to mark a task "Done."

---

Q: What is the "nothing to do" signal?

A: All agents choose "skip" in their response and no new comment is added to the task.

---

Q: If an agent adds a comment, how does the loop proceed?

A: Continue to the next agent immediately, but pass the updated task data (including the new comment) to that next agent. After all agents finish the current pass, the runner retriggers the loop again with the updated task state.

---

Q: When retriggered, where does the loop start?

A: Always from the first agent in the workspace order.

---

Q: Are agents executed sequentially or in parallel?

A: Sequentially, one agent at a time.

---

Q: Is there a hard cap on iterations?

A: No, there is no hard cap on iteration count or time.

---

Q: What happens if a new task event arrives during an active loop?

A: The runner finishes the current loop, then restarts from agent 1. This follows the task event queue rules described in the spec.

---

## Agent Status Changes

Q: Can agents change task status?

A: Yes, but only by requesting a move to "In Review" via the agent response format. No other status changes are allowed from agents.

---

Q: If an agent comments and requests "In Review," what happens?

A: The runner stops the current loop immediately. If the runner already asserted task properties before calling the agent, it should still stop right away. The next queue item will be picked up, but no agents will run because the task is already "In Review."

---

Q: Do agents work on tasks already in "In Review"?

A: No, agents are not allowed to run on tasks in "In Review."

---

## Error Handling

Q: How are agent failures/timeouts handled?

A: The runner writes a System comment with detailed error/timeout info, emits a new task event into the queue, and stops the loop. A new loop will be triggered through the queue (contextual retry forever), without moving the task to "In Review."

---

Q: Is there backoff for retries?

A: No explicit retry or backoff; the task event queue provides natural delay.

---

## Comments

Q: Should system comments be structured or free text?

A: Free text only.

---

Q: What fields should a comment have?

A:
- `task_id: string` - The task this comment belongs to
- `workspace_id: string` - The workspace this comment belongs to
- `user_id?: string` - NULL if system or agent; if present, always `000000000000000000000` (21 zeros, a mock nanoid)
- `agent_id?: string` - NULL if system or user; contains the agent's nanoid if from an agent
- `content: string` - Long markdown string
- `created_at: Date`
- `updated_at: Date`

---

Q: How should `user_id` and `agent_id` be interpreted?

A: `user_id` is NULL for system/agent comments; if present it is always the mock nanoid of all zeros. `agent_id` is NULL for system/user comments. `content` is long markdown text.

---

Q: Will there be multiple users later?

A: Yes, but for now there is only one user with no auth, so the single mock user ID is used everywhere.

---

Q: Should the mock `user_id` be a config value?

A: Hardcoded is acceptable for now.

---

## Agent Configuration

Q: How are agents identified?

A: Each agent has its own nanoid. The default example agents are auto-created per workspace, but users can edit, delete, add, and reorder them.

---

Q: Is an explicit agent `order` field required?

A: Yes, and it must be unique within the workspace. When reordering agents, the client/UI must send all the new valid ordering.

---

## Agent Response Format

Q: What is the agent response format?

A: Agents respond with a JSON structure containing an array of actions. This allows multiple actions in a single response (e.g., comment AND request "In Review").

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

---

Q: What action types are available for agents?

A:
- `skip` - Agent has nothing to do
- `comment` - Agent did meaningful work and has progress to report
- `change_status` - Agent requests to move the task (only "in_review" is allowed)

---

Q: What are the valid action combinations?

A:
1. `skip` alone - agent has nothing to do
2. `comment` alone - agent did work, loop continues
3. `comment` + `change_status: "in_review"` - agent comments and requests human attention
4. `change_status: "in_review"` alone - technically valid but discouraged; agents should always explain why human attention is needed

---

Q: Should agents comment "I have nothing to do"?

A: No. If an agent has nothing to do, it should use `skip`. Never comment "I have nothing to do" or similar.
