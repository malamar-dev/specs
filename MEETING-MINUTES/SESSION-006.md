# Session 006

Please read [SKILL.md](../.claude/skills/explore-specs/SKILL.md) to understand the process used to create this file.

## Question #1: Task Creation and Runner Race Condition

The spec mentions that when a task is in "Todo" status and a queue item is picked up, the task moves to "In Progress". But what happens if a user creates a task and the runner picks it up before the user finishes adding context via comments? Is there a way to create a task in a "draft" mode, or should users compose everything in the description field before creating?

### Answer

This is not an expected action. When creating a new task, the user must fill in all the context into the DESCRIPTION of the task, not by following comments. The user's comment is ONLY being used to steer the agents later on.

## Question #2: Priority Override and Running Task

For the "Prioritize" action on tasks - the spec says it sets `is_priority: true` on the queue item and `false` on all others in the workspace. What happens if a user prioritizes a task that's currently in "Todo" but another task is already mid-loop in "In Progress"? Does the prioritized task wait until the current loop finishes, or does it interrupt the running task?

### Answer

The prioritized task WILL wait until the current loop finishes, not cancelling/interrupting the running task. If actually in a rush situation, the user would prioritize the task needed, then find and "Cancel" the task which is currently being in the loop.

## Question #3: Working Directory Conflicts

The spec mentions that agents share the same working directory and can "indirectly share working resources with each other." For the static working directory scenario, what happens if two different workspaces are configured to use the same directory path? Could agents from different workspaces interfere with each other's work, or should Malamar prevent this configuration?

### Answer

No, it's not Malamar's business to prevent this. Users are responsible for avoiding conflicts if they point multiple workspaces at the same path.

## Question #4: Invalid Agent Actions

The spec describes the agent response format with actions like `skip`, `comment`, and `change_status`. What happens if an agent returns an invalid action type (e.g., `{ "type": "approve" }` or `{ "type": "move_to_done" }`)?

### Answer

This is covered under error handling. An invalid action type falls under "wrong structure" / "doesn't match the expected schema" - treated as malformed output:
1. System comment with detailed error message
2. Stop the current loop immediately
3. The comment emits a task event → new queue item
4. New loop is triggered via that queue item

## Question #5: CLI Status States

For the CLI auto-detection that runs on startup and every 5 minutes - the spec says it runs a test prompt like "Respond with OK". If a CLI is rate-limited or temporarily unavailable during the health check, it would be marked as "Test Failed". Should there be a distinction between "not installed" vs "temporarily unreachable"?

### Answer

Simplified to just two states: "Healthy" and "Unhealthy". Merge "not found" and "test failed" into "unhealthy" to treat them identically. An optional "error message" field in the CLI status object provides more detail (e.g., "binary not found in PATH" or "test prompt returned empty response").

## Question #6: Email Notification Failures

The spec says email notifications are "fire-and-forget async calls" that don't block the runner. If Mailgun is misconfigured or the API call fails, does the failure get logged anywhere?

### Answer

Silent fail for now, only error log of the application is enough. No user-facing alert.

## Question #7: Manual Status Transition and Queue

What if the task was already in "In Progress" because the user moved it there manually? For example, user creates task, manually moves to "In Progress" via some action, but no queue item exists yet. Is this a valid state?

### Answer

Moving a task to "In Progress" or "Todo" always creates a queue item if one doesn't exist. This ensures the runner always picks up tasks that need processing.

## Question #8: Cancel and Child Processes

For the "Cancel" action that sends SIGTERM to the CLI subprocess - what happens if the CLI process has spawned child processes?

### Answer

Malamar is only responsible for the CLI itself. The CLI's subprocesses are its own responsibility to gracefully shutdown - Malamar can't control these.

## Question #9: Done Task Column Performance

For workspaces with many tasks (say, 50+ tasks in the "Done" column over time), could the board become slow or cluttered? Should there be pagination, archiving, or a limit on how many tasks are displayed per column?

### Answer

1. No pagination on Todo, In Progress, In Review at all - show all tasks
2. Adding a "Delete all Done tasks" action to allow quick workspace cleanup
3. Adding an option to workspace settings to automatically delete "Done" tasks after a configurable period:
   - Default: ON, 7 days retention
   - Configurable retention period (in days)
   - Set to 0 to disable auto-deletion (keep forever)

## Question #10: Done Task Auto-Delete Cascade

For the auto-delete of Done tasks - should this also delete the associated activity logs and comments?

### Answer

Yes. Every task when being deleted should cascade all the logs, comments, queue items, etc. This applies to manual deletion, "Delete all Done tasks", and the auto-delete job.

## Question #11: Done Task Cleanup Background Job

Should the auto-delete of Done tasks be a background job?

### Answer

Yes, running on a scheduled basis (daily).

## Question #12: Cleanup Job Consolidation

Should Queue Cleanup and Done Task Cleanup be separate jobs?

### Answer

No, merge them into one "Cleanup" background job that runs daily and handles:
1. Delete `completed` and `failed` queue items older than 7 days
2. Delete Done tasks past their retention period (per workspace setting)

## Question #13: Delete All Done Tasks Confirmation

The "Delete all Done tasks" action - should this require a confirmation step similar to deleting a workspace?

### Answer

Yes, typing the workspace name for confirmation, as this is a many-destructive action.

## Question #14: Workspace Settings Page

Should there be a dedicated "Workspace Settings" page/section?

### Answer

Yes. The Workspace Settings page should have these sections:

1. **General**
   - Title (for the user)
   - Description/Instruction (for agents - the workspace-level context)

2. **Working Directory**
   - Mode: Static Directory or Temp Folder (default)
   - Directory path (if static mode selected)

3. **Task Cleanup**
   - Auto-delete Done tasks: ON/OFF (default: ON)
   - Retention period in days (default: 7, set to 0 to disable)

4. **Notifications**
   - On error occurred: ON/OFF (inherits global default)
   - On task moved to In Review: ON/OFF (inherits global default)

## Question #15: Global Task Cleanup Default

Should there be a global default for task cleanup settings?

### Answer

No, 7 days is hardcoded as the default for new workspaces. No global settings for auto-delete feature. Each workspace manages its own cleanup settings independently.

## Question #16: Tech Stack

Is there a preferred tech stack for the implementation?

### Answer

1. **Backend**: TypeScript with Bun - RESTful API, background jobs, SSE endpoint, SQLite database
2. **Frontend**: TypeScript with React (via create-vite-app) with Bun

## Question #17: Single Executable Distribution

For the single executable distribution - should the frontend be bundled into the backend binary?

### Answer

Yes, if possible.

**Preferred approach**: Bundle the frontend build output directly into the Bun binary if `bun build --compile` supports serving embedded static assets.

**Fallback approach**:
1. Compress the UI dist into the binary
2. On startup, decompress to `~/.malamar/ui/` (or similar)
3. Serve static files from there
4. Overwrite on each startup to ensure UI matches the binary version

## Question #18: Host and Port Configuration

Should the port be configurable?

### Answer

Not just the port, but also the host. Both can be controlled via ENV VAR and CLI flag, with ENV VAR taking priority.

- `MALAMAR_HOST` / `--host` (default: `127.0.0.1`)
- `MALAMAR_PORT` / `--port` (default: `3456`)

## Question #19: Application Configuration Options

What configuration options should be available at the application level?

### Answer

**Server**
- `MALAMAR_HOST` / `--host` - Bind address (default: `127.0.0.1`)
- `MALAMAR_PORT` / `--port` - Server port (default: `3456`)

**Data & Storage**
- `MALAMAR_DATA_DIR` / `--data-dir` - Data directory (default: `~/.malamar`)

**Logging**
- `MALAMAR_LOG_LEVEL` / `--log-level` - Log verbosity: debug, info, warn, error (default: `info`)
- `MALAMAR_LOG_FORMAT` / `--log-format` - Output format: text, json (default: `text`)

**Runner**
- `MALAMAR_RUNNER_POLL_INTERVAL` / `--runner-poll-interval` - Sleep between queue checks in ms (default: `1000`)

**Temp Files**
- `MALAMAR_TEMP_DIR` / `--temp-dir` - Override temp directory for input/output files (default: system `/tmp`)

## Question #20: CLI Commands

What CLI commands should Malamar have?

### Answer

1. `malamar` - Start the server (default)
2. `malamar version` - Show version info
3. `malamar help` - Show help/usage
4. `malamar doctor` - Check system health (CLIs, database, config)
5. `malamar config` - Show current configuration
6. `malamar export` - Export data (to be detailed in future session)
7. `malamar import` - Import data (to be detailed in future session)
