# Workspace Configuration

This guide helps you configure workspaces effectively for different use cases.

---

## Working Directory Modes

Every workspace has a working directory where agents execute their work. Choose between two modes:

### Static Directory Mode

**What it is:** All agents work in a specific directory you specify (e.g., a cloned repository).

**When to use:**
- Working on a single repository
- Agents need access to existing code, configuration, or files
- You want agents to make persistent changes
- Git operations are part of the workflow (commit, push, create PR)

**Example use cases:**
- Maintaining a specific project
- Code review workflows
- Documentation updates to an existing docs folder

**Configuration:**
- Working Directory Mode: Static
- Working Directory Path: `/path/to/your/repo`

**Important considerations:**
- Agents share the directory - changes by one agent are visible to the next
- Multiple workspaces pointing to the same directory can conflict
- The directory must exist and be accessible

### Temp Folder Mode (Default)

**What it is:** Each task gets a fresh, empty temporary directory.

**When to use:**
- Tasks that span multiple repositories
- Tasks that need isolation from each other
- Generating new projects or files from scratch
- Situations where a clean slate is preferred

**Example use cases:**
- Cross-repository dependency updates
- Generating new projects from templates
- Tasks that shouldn't affect a real codebase

**Configuration:**
- Working Directory Mode: Temp Folder
- (No path needed)

**Important considerations:**
- Agents must clone or create files as part of their work
- Include repository URLs in task descriptions so agents know what to clone
- Temporary directories are cleaned up by the OS

---

## Task Cleanup Settings

Control how completed tasks are managed.

### Auto-Delete Done Tasks

**What it is:** Automatically delete tasks in "Done" status after a retention period.

**Options:**
- Enabled (default): Done tasks are deleted after the retention period
- Disabled: Done tasks are kept forever

**When to enable:**
- You don't need historical records of completed tasks
- You want to keep the workspace clean
- Storage space is a concern

**When to disable:**
- You need to reference completed tasks later
- Audit trail is important
- Tasks contain valuable information you might need

### Retention Period

**What it is:** How long to keep Done tasks before auto-deletion.

**Default:** 7 days

**Recommendations:**
- Short (1-3 days): High-volume workflows, tasks are routine
- Medium (7-14 days): Most use cases, balance of cleanliness and reference
- Long (30+ days): Need to reference historical tasks
- Disable (0): Never auto-delete, manage manually

---

## Notification Settings

Control when Malamar sends email notifications.

### On Error Occurred

**What it triggers:** CLI failure, timeout, or malformed output during task processing.

**When to enable:**
- Running tasks autonomously (away from keyboard)
- Want immediate awareness of problems
- CLIs are known to be flaky

**When to disable:**
- Actively monitoring the web UI
- Errors are expected during development/testing
- Too many notifications become noise

### On Task Moved to In Review

**What it triggers:** A task transitions to "In Review" status (agents finished or need human input).

**When to enable:**
- Running tasks while away (mobile via Tailscale)
- Want to know when work is ready for review
- Asynchronous workflow where you check in periodically

**When to disable:**
- Actively monitoring the web UI
- High-volume workflows where constant notifications are disruptive

---

## Workspace Instruction Best Practices

The workspace instruction provides context to all agents.

### What to Include

**Project context:**
```
This workspace manages the Acme API, a REST service for customer management.
The codebase uses TypeScript, Express, and PostgreSQL.
```

**Workflow guidance:**
```
Agents work together to plan, implement, review, and approve changes.
Communicate through comments, referencing other agents by name.
```

**Repository information (for temp folder mode):**
```
The main repository is at https://github.com/acme/api
Clone it when working on implementation tasks.
```

**Standards and conventions:**
```
Follow the existing code style. Use ESLint and Prettier.
All new functions must have JSDoc comments.
Tests are required for new features.
```

### What NOT to Include

**Agent-specific instructions:** Put these in the agent's own instruction, not the workspace instruction.

**Task-specific details:** Put these in the task description.

**Temporary notes:** Use task comments for in-progress communication.

### Example Workspace Instruction

```
This workspace manages the Acme Web App, a React application for customer self-service.

Repository: https://github.com/acme/webapp
Tech Stack: React 18, TypeScript, TailwindCSS, Vite

Agents collaborate as follows:
- Planner creates implementation plans based on task requirements
- Implementer writes code following the plan
- Reviewer ensures code quality and catches issues
- Approver verifies the work meets requirements

Standards:
- Follow existing component patterns in src/components
- Use hooks for state management
- All components need corresponding test files
- Commit messages follow conventional commits format

When tasks require UI changes, include screenshots or mockups in the description.
```

---

## Configuration Recipes

### Recipe: Single Repository Maintenance

**Use case:** Ongoing development on one codebase

```
Working Directory Mode: Static
Working Directory Path: /home/user/projects/myapp
Auto-Delete Done Tasks: Enabled, 7 days
Notifications: Both enabled
```

### Recipe: Multi-Repository Operations

**Use case:** Tasks that span multiple repos

```
Working Directory Mode: Temp Folder
Auto-Delete Done Tasks: Enabled, 3 days
Notifications: Both enabled
```

Workspace instruction should list repositories:
```
This workspace handles cross-repo updates.
Repositories:
- https://github.com/org/frontend
- https://github.com/org/backend
- https://github.com/org/shared-lib
```

### Recipe: Content Generation

**Use case:** Generating blog posts, documentation, etc.

```
Working Directory Mode: Temp Folder
Auto-Delete Done Tasks: Disabled (keep for reference)
Notifications: On In Review only (less urgent)
```

---

## Tips for the Malamar Agent

When helping users configure workspaces:

1. **Ask about their use case** - Single repo or multiple? Persistent or ephemeral?
2. **Recommend temp folder for beginners** - Safer, simpler to start
3. **Check path accessibility** - Static paths must exist
4. **Consider notification needs** - Are they running autonomously or monitoring?
5. **Suggest workspace instruction content** - Help them write effective context
