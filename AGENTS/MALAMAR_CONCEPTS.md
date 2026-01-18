# Malamar Concepts

This guide explains the foundational concepts of how Malamar works.

---

## The Multi-Agent Loop

The multi-agent loop is how Malamar orchestrates agents to work on tasks autonomously.

### How It Works

```
┌─────────────────────────────────────────────────┐
│                                                 │
│  Task Picked Up                                 │
│       ↓                                         │
│  Agent 1 (Planner) → skip/comment/in_review     │
│       ↓                                         │
│  Agent 2 (Implementer) → skip/comment/in_review │
│       ↓                                         │
│  Agent 3 (Reviewer) → skip/comment/in_review    │
│       ↓                                         │
│  Agent 4 (Approver) → skip/comment/in_review    │
│       ↓                                         │
│  End of Pass                                    │
│       ↓                                         │
│  Any comments added? ──Yes──→ Restart from Agent 1
│       │                                         │
│       No                                        │
│       ↓                                         │
│  Move to "In Review"                            │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Key Behaviors

1. **Sequential Execution:** Agents run one at a time in order
2. **Fresh Context:** Each agent sees the latest task state, including comments from previous agents in the same pass
3. **Loop Restart:** If any agent adds a comment, the loop restarts from Agent 1
4. **Termination:** The loop ends when:
   - All agents skip (no one has work to do)
   - Any agent requests "In Review" (human attention needed)
   - An error occurs (task stays in progress, retry happens)

### Why This Design?

- **Self-correcting:** Later agents catch mistakes earlier agents make
- **Fresh perspective:** Each agent starts with clean context
- **Natural checkpoints:** Work progresses through review stages automatically

---

## Task Lifecycle

Tasks move through four statuses:

```
┌────────┐     ┌─────────────┐     ┌───────────┐     ┌──────┐
│  Todo  │ ──→ │ In Progress │ ──→ │ In Review │ ──→ │ Done │
└────────┘     └─────────────┘     └───────────┘     └──────┘
     ↑               │                   │               │
     └───────────────┴───────────────────┴───────────────┘
                    (can move backward)
```

### Status Meanings

| Status | What's Happening | Who Acts |
|--------|-----------------|----------|
| **Todo** | Waiting in queue | Runner picks it up |
| **In Progress** | Agents actively working | Agents |
| **In Review** | Needs human attention | Human user |
| **Done** | Completed | No one (archived) |

### Transitions

- **Todo → In Progress:** Runner picks up the task
- **In Progress → In Review:** Agents finished OR agent requested human attention OR error needs human review
- **In Review → Todo:** User comments (restarts agent processing)
- **In Review → Done:** User marks complete
- **Done → Todo:** User reopens the task

### Important Notes

- Only one task per workspace is "In Progress" at a time
- Agents can only request "In Review" - they cannot move tasks to "Done"
- Moving backward is always possible for flexibility

---

## Agent Actions

When an agent finishes working on a task, it responds with actions:

### Skip

**What it means:** "I have nothing to do right now."

**When to use:**
- The task isn't ready for this agent yet
- This agent's work is already done
- Waiting for another agent to act first

**Effect:** Agent passes, loop continues to next agent

### Comment

**What it means:** "I did meaningful work and have something to report."

**When to use:**
- Created a plan or made progress
- Providing feedback to another agent
- Asking a question or raising a concern

**Effect:** Comment is added, loop will restart after all agents finish

### Change Status (to In Review)

**What it means:** "This task needs human attention."

**When to use:**
- Work is complete and ready for human review
- Blocked by something that requires human decision
- Found an issue that needs human judgment

**Effect:** Loop stops immediately, task moves to "In Review"

### Combinations

Agents can combine actions:
- **Comment + Change Status:** Report what was done AND request human review (common for the Approver)
- **Skip only:** Nothing to do
- **Comment only:** Progress report, loop will continue

---

## Comments as Communication

Comments are the primary way stakeholders communicate:

### Comment Authors

| Author | When Used |
|--------|-----------|
| **User** | Human adds input to steer agents |
| **Agent** | Agent reports progress, feedback, or questions |
| **System** | Malamar reports errors, status changes, or notifications |

### How Agents Use Comments

1. **Read previous comments** to understand what happened
2. **Address other agents by name** for directed feedback
3. **Add their own comment** to report work or provide feedback

### Comment Flow Example

```
[User] Created task: "Add user authentication"
[Planner] ## Implementation Plan
          1. Add login endpoint
          2. Add session management
          3. Add protected route middleware
[Implementer] ## Implementation Progress
              Completed steps 1-3. Added tests.
[Reviewer] Implementer, the session token is stored insecurely.
           Please use httpOnly cookies instead of localStorage.
[Implementer] ## Implementation Progress
              Fixed: Now using httpOnly cookies for session tokens.
[Reviewer] ## Code Review
           Status: Approved
[Approver] ## Ready for Review
           Authentication complete and reviewed.
```

---

## Activity Logs

Activity logs track events that happen to a task:

### Event Types

| Event | Meaning |
|-------|---------|
| `created` | Task was created |
| `status_changed` | Status transitioned |
| `agent_started` | Agent began working on task |
| `agent_finished` | Agent completed their pass |
| `comment_added` | A comment was added |

### Difference from Comments

- **Comments:** Content for communication (markdown, prose)
- **Activity Logs:** Structured events for audit trail (timestamps, actors)

Both are included in the context passed to agents, but they serve different purposes.

---

## Queue Priority

When multiple tasks are in "Todo", the runner picks them up in a specific order:

### Priority Order

1. **Prioritized tasks:** Tasks explicitly prioritized by the user (is_priority flag)
2. **Most recently processed:** The task that was just worked on (focus on finishing one task)
3. **Most recently updated:** LIFO - newer tasks first

### Why This Order?

- **Finish what you start:** Prioritizing recently processed tasks means Malamar tries to complete a task before starting another
- **User control:** Explicit priority lets users bump important tasks
- **Fresh tasks surface:** LIFO means users can "bump" a task by commenting on it

---

## Workspaces

Workspaces are containers for related work:

### What a Workspace Contains

- **Title:** Human-readable name
- **Instruction:** Context for all agents in the workspace
- **Agents:** The team of AI agents configured for this workspace
- **Tasks:** Work items for agents to process
- **Chats:** Conversations with agents for ad-hoc help

### Workspace Philosophy

- **One concern:** Each workspace should focus on one type of work
- **Shared context:** All agents share the workspace instruction
- **Independent queues:** Workspaces process tasks independently of each other

### Multiple Workspaces

- Different workspaces can run in parallel
- Each workspace has its own agents tuned for that type of work
- The same repository can have multiple workspaces (e.g., one for features, one for docs)

---

## Parallel Execution

How Malamar handles concurrency:

### Across Workspaces

- Multiple workspaces can process tasks simultaneously
- Each workspace has its own worker
- No limit on concurrent workspace processing

### Within a Workspace

- Only ONE task is "In Progress" at a time
- Agents execute sequentially (not in parallel)
- This ensures agents see each other's work clearly

### Why Not Parallel Tasks?

- Agents might conflict if working on the same codebase
- Sequential processing ensures clear cause-and-effect
- Simpler mental model for users

---

## Tips for the Malamar Agent

When explaining concepts to users:

1. **Use the diagrams** - Visual representations help understanding
2. **Relate to their situation** - "In your workflow, the Reviewer would..."
3. **Start with the loop** - It's the core concept everything else builds on
4. **Emphasize termination** - Users often wonder "when does it stop?"
5. **Clarify agent vs user control** - What agents can do vs what only humans can do
