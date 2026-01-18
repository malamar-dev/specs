# Session 010

Please read [SKILL.md](../.claude/skills/explore-specs/SKILL.md) to understand the process used to create this file.

**Session Focus:** V1 scope completeness review and UI/UX decisions before moving to detailed technical design and implementation.

---

## Question #1: Sample Workspace Agent Instructions Content

The first startup behavior (SESSION-009 Q#27) specifies that Malamar auto-creates a sample workspace with four agents. However, the actual content of these instructions hadn't been confirmed.

### Answer

The sample agent instructions are already fully defined in `APPENDIX_ONBOARDING_RESOURCES.md`, along with the Malamar agent's hardcoded instruction. The AGENTS/ knowledge base contains comprehensive guidance on writing instructions, workflow patterns, workspace configuration, and troubleshooting.

---

## Question #2: Knowledge Base URL and Offline Scenarios

The Malamar agent instruction references the knowledge base at a GitHub URL. Scenarios like first-time users without internet, private repository visibility, or version mismatches were considered.

### Answer

The current approach is sufficient. The Malamar agent instruction itself contains enough guidance for basic help. The knowledge base is supplementary. The specs repository will be public to serve external users. Silent failure on network issues (as decided in SESSION-008 Q#2) is appropriate for transient issues.

---

## Question #3: V1 Feature Scope Confirmation

A comprehensive review of features to confirm what's in v1 scope.

### Answer

**Core Features (confirmed in v1):**
- Workspaces (CRUD, settings, working directory modes)
- Agents (CRUD, reorder, CLI assignment)
- Tasks (CRUD, statuses, Kanban board)
- Task Comments & Activity Logs
- Multi-agent Loop (runner, queue, error handling)
- CLI Adapters (Claude Code, Gemini CLI, Codex CLI, OpenCode)
- Chat with Agents (workspace agents + Malamar agent)
- Chat Actions (create/update/delete agents, update workspace)
- Chat File Attachments
- Email Notifications (Mailgun)
- Real-time Updates (SSE + polling)
- CLI Settings (paths, env vars, health check)
- First Startup (sample workspace)
- Auto-delete Done Tasks

**CLI Commands in v1:**
- `malamar` (start server)
- `malamar version`
- `malamar help`
- `malamar doctor` (check CLIs, database, config)
- `malamar config` (show current configuration)

**Deferred to ROADMAP:**
- S3 Backup
- Import/Export
- Clone Workspace
- In-Browser Push Notifications
- Related Tasks Query
- Agent Client Protocol (ACP)
- Community Gallery

---

## Question #4: Delete Confirmation Patterns

Different confirmation patterns exist for destructive actions. The pattern needed to be consistent.

### Answer

**Impact-based confirmation pattern:**

| Action | Confirmation |
|--------|--------------|
| Delete Workspace | Type workspace name |
| Delete All Done Tasks | Type workspace name |
| Delete Individual Task | Simple confirm dialog |
| Delete Individual Agent | Simple confirm dialog |
| Delete Individual Chat | Simple confirm dialog |

Type-to-confirm is used for batch/high-impact deletions. Simple confirm is used for single-item deletions.

---

## Question #5: UI Component Framework

The tech stack specifies React but doesn't specify a component library.

### Answer

**shadcn/ui + Tailwind CSS** for the frontend:
- Modern, accessible components copied into the project (not a dependency)
- Highly customizable
- Popular in the React ecosystem with good documentation
- Works well with Vite

---

## Question #6: Mobile Responsiveness

The specs mention Tailscale access for remote/mobile use but don't specify responsiveness requirements.

### Answer

**Mobile-first responsive design** for v1. The UI should be designed mobile-first and scale up to desktop, not the other way around. All features should work well on mobile devices.

---

## Question #7: Search and Filter Functionality

No search or filtering was specified for list views.

### Answer

**Basic text search** for v1:
- Workspace list: Text search by workspace title
- Chat list: Text search by chat title

No advanced filters. No search for tasks (Kanban already filters by status columns). Search can be expanded in future versions based on user feedback.

---

## Question #8: Empty States

When lists are empty, the UI behavior wasn't specified.

### Answer

**Contextual empty states with CTAs:**

| Empty State | Display |
|-------------|---------|
| Workspace list | "No workspaces yet" + "Create Workspace" button |
| Tasks (whole board) | "No tasks yet" + "Create Task" button |
| Tasks (single column) | Empty column, no message |
| Chats | "No conversations yet" + "Start a chat" dropdown |
| Agents | Warning banner (per SESSION-008 Q#11) + "Create Agent" or prompt to chat with Malamar |

---

## Question #9: Form Validation Rules

Validation rules for forms weren't specified.

### Answer

**Minimal validation with required fields only:**

**Required fields:**
- Workspace title
- Agent name
- Agent instruction
- Task summary
- Comment content (non-empty)
- Chat message (non-empty)

**Optional fields:**
- Workspace instruction
- Task description
- Directory path (static mode)

**Uniqueness:**
- Agent name must be unique within workspace

**No max lengths:** As decided in SESSION-004 Q#29

**Path validation:** Don't validate existence at save time (path might be created later). Show warning if path doesn't exist but allow saving.

**UI truncation:** Truncate long text in lists/cards with ellipsis, show full text in detail views.

---

## Question #10: Kanban Task Card Content

What information should task cards display on the Kanban board wasn't fully specified.

### Answer

**Essential info with activity indicator:**
- Task summary (title) - truncated with ellipsis if long
- Time since last update (e.g., "5 min ago", "2 days ago")
- Comment count badge (if > 0)
- Visual indicator if currently being processed by an agent (e.g., subtle pulsing border or spinner)

No description preview on cards - that's shown in the task detail view.

---

## Question #11: Toast Notifications Behavior

SSE events trigger toast notifications but the behavior wasn't specified.

### Answer

**Auto-dismiss with interaction option:**
- Duration: 5 seconds auto-dismiss
- Stacking: Max 3 visible at once, older ones dismissed when new arrive
- Position: Bottom-right
- Clickable: Clicking toast navigates to relevant item (task, chat)
- Dismiss: X button to dismiss early
- Types: Success (green), Error (red), Info (neutral) - matching event severity

---

## Question #12: All CLIs Unhealthy Scenario

What happens when all CLIs are unhealthy wasn't fully specified.

### Answer

**Graceful degradation with clear messaging:**

| Scenario | Behavior |
|----------|----------|
| Task processing | Error handling continues (system comment, retry). Add workspace-level warning banner when all assigned agent CLIs are unhealthy. |
| Chat | Show error in chat UI before sending: "No CLI available. Please configure a CLI in settings." Prevent sending until a CLI is healthy. |
| Agent creation | Show all CLIs in dropdown with visual indicator (red dot for unhealthy). User can still assign - they'll fix the CLI later. |
| Global | Persistent warning in header/nav when zero CLIs are healthy: "No CLIs available - configure in settings" |

---

## Question #13: Task Prioritization UI

The prioritization mechanics were defined but not the UI.

### Answer

**Explicit prioritize/deprioritize with visual indicator:**
- **Button location:** In task detail view, alongside other action buttons
- **Visual indicator:** Prioritized tasks show a badge/icon on the Kanban card (e.g., flag icon or "Priority" label)
- **Toggle behavior:** Button shows "Prioritize" if not prioritized, "Remove Priority" if prioritized
- **Only one:** When prioritizing a task, any other prioritized task in that workspace automatically loses priority (existing behavior)

---

## Question #14: Agent Ordering - Default and Reordering UI

How new agents get ordered and how users reorder agents wasn't specified.

### Answer

**Append to end + drag-to-reorder:**
- **New agent default:** Appends to the end (highest order value + 1)
- **Agent list display:** Shows agents in order with position numbers (1, 2, 3, 4)
- **Reorder UI:** Drag-and-drop in the agent list. Drop saves immediately.
- **Mobile alternative:** Up/down arrow buttons next to each agent (drag can be awkward on touch)

---

## Question #15: Default CLI for New Agents

When creating a new agent, which CLI should be pre-selected wasn't specified.

### Answer

**First healthy CLI in priority order:**
1. Claude Code
2. Codex CLI
3. Gemini CLI
4. OpenCode

Pre-select the first healthy one. If none healthy, pre-select Claude Code anyway (user will see the unhealthy indicator and fix it). Malamar agent actions follow the same logic.

---

## Question #16: Theme Support (Dark/Light Mode)

Theme support wasn't mentioned in the specs.

### Answer

**System preference with toggle:**
- Default: Follow system preference (prefers-color-scheme)
- Toggle: Add theme switcher in global settings or header
- shadcn/ui has built-in dark mode support, making this low effort
- Store preference in localStorage

---

## Question #17: Date/Time Display Format

How timestamps should be formatted wasn't specified.

### Answer

**Relative for recent, absolute for old:**
- < 1 hour: "5 min ago", "just now"
- < 24 hours: "3 hours ago"
- < 7 days: "2 days ago"
- ≥ 7 days: "Jan 15" (current year) or "Jan 15, 2025" (past year)
- Hover tooltip: Show full absolute timestamp (e.g., "January 15, 2025 at 3:45 PM")
- Use browser locale for formatting (respects user's date preferences)
- Times in user's local timezone

---

## Question #18: Scope Completeness Check

Summary of all decisions made in this session was reviewed.

### Answer

All UI/UX decisions for v1 are now documented. A final review of SPECS.md and TECHNICAL_DESIGN.md was requested to ensure they reflect all decisions.

---

## Question #19: Document Issues Found

A review of SPECS.md and TECHNICAL_DESIGN.md identified several issues.

### Answer

**Issues in TECHNICAL_DESIGN.md:**

1. **Outdated reference to `create_task` action (Line 634):** States the `task.created` event is emitted when an agent uses `create_task` from chat. This action was removed in SESSION-009 Q#34-35. The event is still valid for UI-created tasks but the chat reference is outdated.

2. **Export/Import commands listed but deferred (Lines 762-763):** `malamar export` and `malamar import` are listed in CLI commands with "details TBD" but these are in ROADMAP.md as future features, not v1.

**Missing from both documents:**

| Decision | Status |
|----------|--------|
| UI Framework (shadcn/ui + Tailwind CSS) | Not documented |
| Mobile-first responsive design | Not documented |
| Delete confirmation patterns | Not documented |
| Basic search (workspace, chat lists) | Not documented |
| Empty states with CTAs | Not documented |
| Form validation rules | Not documented |
| Kanban card content details | Partially documented |
| Toast behavior (5s, max 3, clickable) | Not documented |
| Unhealthy CLI warning banners | Not documented |
| Task prioritization UI (button, badge) | Not documented |
| Agent ordering UI (drag-to-reorder) | Not documented |
| Default CLI for new agents | Not documented |
| Theme support (dark/light + toggle) | Not documented |
| Date/time display format | Not documented |

**Next steps:** Update documents with all decisions from this session (to be done in a subsequent action).

---

## Post-Session Notes

### Note 1: `task.created` SSE Event Removed

The `task.created` SSE event was originally designed for the `create_task` chat action (agents creating tasks via chat). Since the `create_task` action was removed in SESSION-009 Q#34-35, the `task.created` event has no purpose and has been removed from TECHNICAL_DESIGN.md entirely.

### Note 2: Search Feature Requires API Design

The basic search functionality added in this session (Q#7) affects the API design:
- Workspace list search needs a query parameter on `GET /api/workspaces`
- Chat list search needs a query parameter on `GET /api/workspaces/:id/chats`

**Action required:** Clarify the search API design in a later session and update the API endpoints section in TECHNICAL_DESIGN.md accordingly.
