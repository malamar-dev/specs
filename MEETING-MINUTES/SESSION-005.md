# Session 005

Please read [SKILL.md](../.claude/skills/explore-specs/SKILL.md) to understand the process used to create this file.

## Question #1: Input File Overwrite Concern

The input file is overwritten for each agent execution. Is there any concern about agents needing to reference their own previous work from earlier in the same loop?

### Answer

No concern. All historical data of the task should be in the comments as the source of truth. Additionally, "Activity Logs" provide linear historical tracking. For example:
- User created
- System changed status from "Todo" to "In Progress"
- Agent "Planner" added a comment
- etc.

This keeps track of all task history even though the input file is rewritten every time.

## Question #2: Activity Log Storage

Should Activity Logs be stored as separate database records, embedded within the task record, or derived on-the-fly?

### Answer

Separate database records in a `task_logs` table, alongside the `task_comments` table.

## Question #3: Activity Log Fields

What fields should a `task_logs` entry have?

### Answer

- `task_id`
- `workspace_id`
- `event_type` (e.g., "created", "status_changed", "comment_added", "agent_started", "agent_finished")
- `actor_type` (e.g., "user", "agent", "system")
- `actor_id` (nullable - the user_id or agent_id)
- `metadata` (JSON for event-specific data like old_status/new_status)
- `created_at`

## Question #4: Activity Log Display Location

Should Activity Logs be displayed inline with comments or as a separate tab?

### Answer

Separate "Activity" tab in the task detail popup.

## Question #5: Task UI Flow

What is the UI flow for accessing tasks?

### Answer

1. When accessing a "Workspace", see a Kanban board of tasks (similar to JIRA, Trello, or Linear)
2. When selecting a task, a popup displays the task's detail
3. Layout similar to JIRA or Linear: title, markdown description, then a separated section with tab selection for comments/activity log (dropdown for mobile)
4. Comments and activity logs displayed in DESC created_at order (newest on top)

## Question #6: Comment Display Order Rationale

For comments, wouldn't newest-on-top make agent conversations harder to follow?

### Answer

In the UI, display in DESC order for easy catch-up on recent changes. But in the input file, list all comments/activity logs in ASC order for agents to read the conversation in natural chronological flow.

## Question #7: Activity Logs in Input File

Should activity logs be included in the input file passed to agents?

### Answer

Yes, for now. Start inclusive and trim later if it adds noise rather than meaningful context.

## Question #8: Activity Logs Section in Input File

Should activity logs be in a separate section from comments in the input file, or interleaved?

### Answer

Separate sections:
```markdown
## Comments
{comments in ASC order}

## Activity Log
{activity logs in ASC order}
```

## Question #9: Kanban Column Mapping

Should the four task statuses map directly to four Kanban columns?

### Answer

Yes, four title-hardcoded columns mapping directly to Todo, In Progress, In Review, Done. No grouping or customization for now.

## Question #10: Task Ordering in Columns

What should the default ordering be for tasks within a column?

### Answer

Most recently updated on top.

## Question #11: Drag-and-Drop Status Change

Should users be able to drag tasks between columns to change status?

### Answer

No, only allow status changes via action buttons in the task detail popup.

## Question #12: Edit vs Comment Status Transition

Should editing task title/description trigger auto-transition from "In Review" to "In Progress"?

### Answer

No, only comment action triggers this. User can manually edit the description, then make a new comment to share context with agents, and the task transfers to "In Progress" automatically following the runner's picking rules.

## Question #13: Task Action Buttons Per Status

What action buttons should be available for each status?

### Answer

**Todo:**
- "Delete" action
- "Prioritize" action (force task to top of queue)
- NO "Move to In Progress" - if a task is running, this task will be transferred back to "Todo" by the rule of one task processed at a time

**In Progress:**
- "Cancel" action (kills current loop)
- "Move to In Review" action
- "Prioritize" action
- NO "Delete" action

**In Review:**
- "Move to Todo" action
- "Delete" action

**Done:**
- "Move to Todo" action
- "Delete" action

## Question #14: Cancel vs Kill Loop

Is "Cancel" the same as "Kill Loop"?

### Answer

Yes. Two use cases:
1. Un-hang the agent/CLI when something went wrong
2. Make time to rethink

The Cancel action kills the current loop but does NOT automatically change the status. The task stays in "In Progress" and the user decides what to do next.

## Question #15: Post-Cancel Behavior

After "Cancel", will the runner automatically pick up the task again?

### Answer

Yes, intended to be re-picked up right away following runner rules. If user doesn't want it picked up anymore, they manually move it to "In Review".

## Question #16: Queue Pickup Status Filter

How should queue pickup handle tasks in "In Review" or "Done" status?

### Answer

Revised pickup logic:

1. **Status filter first**: Only consider queue items where the associated task has status "Todo" or "In Progress"
2. **Then apply priority order** (within filtered items):
   - Any queue item with `is_priority: true` and status `queued`
   - Task that most recently has a `completed` or `failed` queue item → pick its `queued` item
   - Most recently updated `queued` item (LIFO by `updated_at`)

Queue items for tasks in "In Review" or "Done" are simply ignored until the status changes back. This prevents wasted runner cycles.

## Question #17: Alternative Notification Channels

Have you considered notification channels other than Mailgun?

### Answer

Mailgun as the only one for now. The next one should be in-browser push notifications.

## Question #18: Push Notification Events

Should in-browser push notifications have different events than email?

### Answer

No, keep it simple - same events as email notifications.

## Question #19: Workspace Cloning

Would you want the ability to duplicate/clone workspaces?

### Answer

Yes, clone workspace/agent/task are nice features.

## Question #20: Clone Workspace Contents

When cloning a workspace, should it include tasks?

### Answer

No, clone just the workspace settings + agents (clean slate for tasks). Clone the "template" but start fresh with tasks.

## Question #21: Workspace List Page Design

What should the workspace list/home page look like?

### Answer

A detailed list where each row is a card of a workspace, showing:
- Name
- Number of agents
- Number of tasks in Todo/In Progress/In Review (Done count not needed on listing page)

List ordering options:
1. Recently has activity (default) - requires `last_activity_at` column updated via event listener
2. DESC/ASC by created_at
3. DESC/ASC by updated_at

## Question #22: Events Updating last_activity_at

What events should update `last_activity_at`?

### Answer

All tracked events (task created, status changed, comment added, agent execution started/finished, etc.) - any activity indicates the workspace is "alive".

## Question #23: Database Migration Handling

How should database migrations be handled when Malamar is updated?

### Answer

The application should run migrations on startup and panic if migration fails, to ensure consistency. Also open to exploring other schema-free embedded databases for later consideration.

## Question #24: Research Items

Are there any research items identified during this session?

### Answer

Investigate Agent Client Protocol (ACP, https://zed.dev/acp) as an alternative to calling CLIs via subprocess commands.
