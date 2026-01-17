# Session 005

## Activity Logs

Q: The input file is overwritten for each agent execution. Is there any concern about agents needing to reference their own previous work from earlier in the same loop?

A: No, all historical data of the task should be in the comments as the source of truth. Additionally, "Activity Logs" provide linear historical tracking. For example:
- User created
- System changed status from "Todo" to "In Progress"
- Agent "Planner" added a comment
- etc.

This keeps track of all task history even though the input file is rewritten every time.

---

Q: Should Activity Logs be stored as separate database records, embedded within the task record, or derived on-the-fly?

A: Separate database records in a `task_logs` table, alongside the `task_comments` table.

---

Q: What fields should a `task_logs` entry have?

A:
- `task_id`
- `workspace_id`
- `event_type` (e.g., "created", "status_changed", "comment_added", "agent_started", "agent_finished")
- `actor_type` (e.g., "user", "agent", "system")
- `actor_id` (nullable - the user_id or agent_id)
- `metadata` (JSON for event-specific data like old_status/new_status)
- `created_at`

---

Q: Should Activity Logs be displayed inline with comments or as a separate tab?

A: Separate "Activity" tab in the task detail popup.

---

## UI Layout and Navigation

Q: What is the UI flow for accessing tasks?

A:
1. When accessing a "Workspace", see a Kanban board of tasks (similar to JIRA, Trello, or Linear)
2. When selecting a task, a popup displays the task's detail
3. Layout similar to JIRA or Linear: title, markdown description, then a separated section with tab selection for comments/activity log (dropdown for mobile)
4. Comments and activity logs displayed in DESC created_at order (newest on top)

---

Q: For comments, wouldn't newest-on-top make agent conversations harder to follow?

A: In the UI, display in DESC order for easy catch-up on recent changes. But in the input file, list all comments/activity logs in ASC order for agents to read the conversation in natural chronological flow.

---

Q: Should activity logs be included in the input file passed to agents?

A: Yes, for now. Start inclusive and trim later if it adds noise rather than meaningful context.

---

Q: Should activity logs be in a separate section from comments in the input file, or interleaved?

A: Separate sections:
```markdown
## Comments
{comments in ASC order}

## Activity Log
{activity logs in ASC order}
```

---

## Kanban Board

Q: Should the four task statuses map directly to four Kanban columns?

A: Yes, four title-hardcoded columns mapping directly to Todo, In Progress, In Review, Done. No grouping or customization for now.

---

Q: What should the default ordering be for tasks within a column?

A: Most recently updated on top.

---

Q: Should users be able to drag tasks between columns to change status?

A: No, only allow status changes via action buttons in the task detail popup.

---

## Task Status Transitions

Q: Should editing task title/description trigger auto-transition from "In Review" to "In Progress"?

A: No, only comment action triggers this. User can manually edit the description, then make a new comment to share context with agents, and the task transfers to "In Progress" automatically following the runner's picking rules.

---

## Task Action Buttons

Q: What action buttons should be available for each status?

A:

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

---

Q: Is "Cancel" the same as "Kill Loop"?

A: Yes. Two use cases:
1. Un-hang the agent/CLI when something went wrong
2. Make time to rethink

The Cancel action kills the current loop but does NOT automatically change the status. The task stays in "In Progress" and the user decides what to do next.

---

Q: After "Cancel", will the runner automatically pick up the task again?

A: Yes, intended to be re-picked up right away following runner rules. If user doesn't want it picked up anymore, they manually move it to "In Review".

---

## Queue Pickup Rules (Revised)

Q: How should queue pickup handle tasks in "In Review" or "Done" status?

A: Revised pickup logic:

1. **Status filter first**: Only consider queue items where the associated task has status "Todo" or "In Progress"
2. **Then apply priority order** (within filtered items):
   - Any queue item with `is_priority: true` and status `queued`
   - Task that most recently has a `completed` or `failed` queue item → pick its `queued` item
   - Most recently updated `queued` item (LIFO by `updated_at`)

Queue items for tasks in "In Review" or "Done" are simply ignored until the status changes back. This prevents wasted runner cycles.

---

## Notifications

Q: Have you considered notification channels other than Mailgun?

A: Mailgun as the only one for now. The next one should be in-browser push notifications.

---

Q: Should in-browser push notifications have different events than email?

A: No, keep it simple - same events as email notifications.

---

## Workspace Features

Q: Would you want the ability to duplicate/clone workspaces?

A: Yes, clone workspace/agent/task are nice features.

---

Q: When cloning a workspace, should it include tasks?

A: No, clone just the workspace settings + agents (clean slate for tasks). Option 1: clone the "template" but start fresh with tasks.

---

## Workspace List Page

Q: What should the workspace list/home page look like?

A: A detailed list where each row is a card of a workspace, showing:
- Name
- Number of agents
- Number of tasks in Todo/In Progress/In Review (Done count not needed on listing page)

List ordering options:
1. Recently has activity (default) - requires `last_activity_at` column updated via event listener
2. DESC/ASC by created_at
3. DESC/ASC by updated_at

---

Q: What events should update `last_activity_at`?

A: All tracked events (task created, status changed, comment added, agent execution started/finished, etc.) - any activity indicates the workspace is "alive".

---

## Database and Migrations

Q: How should database migrations be handled when Malamar is updated?

A: The application should run migrations on startup and panic if migration fails, to ensure consistency. Also open to exploring other schema-free embedded databases for later consideration.

---

## Research Items

- Investigate Agent Client Protocol (ACP, https://zed.dev/acp) as an alternative to calling CLIs via subprocess commands.
