# Malamar Technical Design

This document provides the architectural overview of Malamar - the big picture of how components fit together, key design decisions, and cross-cutting concerns.

For product specifications (what and why), see [SPECS.md](./SPECS.md).

---

## Table of Contents

- [System Architecture](#system-architecture)
- [Detailed Design Documents](#detailed-design-documents)
- [Key Architectural Decisions](#key-architectural-decisions)
- [Cross-Cutting Concerns](#cross-cutting-concerns)

---

## System Architecture

### High-Level Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              Malamar Binary                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                 │
│  │   Web UI    │    │  REST API   │    │     SSE     │                 │
│  │   (React)   │◄──►│  Endpoints  │    │  Endpoint   │                 │
│  └─────────────┘    └──────┬──────┘    └──────┬──────┘                 │
│                            │                   │                        │
│                            ▼                   │                        │
│  ┌─────────────────────────────────────────────┴───────────────────┐   │
│  │                        Service Layer                             │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │  Workspace  │  │    Task     │  │    Chat     │              │   │
│  │  │   Service   │  │   Service   │  │   Service   │              │   │
│  │  └─────────────┘  └─────────────┘  └─────────────┘              │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                            │                                            │
│                            ▼                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                     Background Jobs                              │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │   │
│  │  │   Runner    │  │   Cleanup   │  │ CLI Health  │              │   │
│  │  │  (per WS)   │  │    Job      │  │   Check     │              │   │
│  │  └──────┬──────┘  └─────────────┘  └─────────────┘              │   │
│  └─────────┼───────────────────────────────────────────────────────┘   │
│            │                                                            │
│            ▼                                                            │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      CLI Adapters                                │   │
│  │  ┌────────┐  ┌────────┐  ┌────────┐  ┌──────────┐               │   │
│  │  │ Claude │  │ Gemini │  │ Codex  │  │ OpenCode │               │   │
│  │  └────┬───┘  └────┬───┘  └────┬───┘  └────┬─────┘               │   │
│  └───────┼───────────┼───────────┼───────────┼─────────────────────┘   │
│          │           │           │           │                          │
│          ▼           ▼           ▼           ▼                          │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    Subprocess Execution                          │   │
│  │         (Input File → CLI Process → Output File)                 │   │
│  └─────────────────────────────────────────────────────────────────┘   │
│                                                                         │
├─────────────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐                 │
│  │   SQLite    │    │  Event Bus  │    │  Temp Files │                 │
│  │  Database   │    │  (In-Proc)  │    │   (/tmp)    │                 │
│  └─────────────┘    └─────────────┘    └─────────────┘                 │
└─────────────────────────────────────────────────────────────────────────┘
```

### Component Relationships

```
Workspace
    │
    ├── Agents (ordered, 0..n)
    │       │
    │       └── CLI Assignment (claude|gemini|codex|opencode)
    │
    ├── Tasks (0..n)
    │       │
    │       ├── Comments (user|agent|system)
    │       ├── Activity Logs
    │       └── Queue Items
    │
    └── Chats (0..n)
            │
            ├── Messages (user|agent|system)
            └── Queue Items
```

### Data Flow: Task Processing

```
1. User creates task
         │
         ▼
2. Task event emitted → Queue item created
         │
         ▼
3. Runner picks up queue item
         │
         ▼
4. For each agent (by order):
         │
         ├── Create input file (markdown)
         │         │
         │         ▼
         ├── Invoke CLI subprocess
         │         │
         │         ▼
         ├── Read output file (JSON)
         │         │
         │         ▼
         └── Execute actions (skip|comment|change_status)
                   │
                   ▼
5. If comment added → Retrigger from agent 1
   If all skip → Move to "In Review"
   If change_status → Stop, move to "In Review"
```

---

## Detailed Design Documents

| Document | Scope |
|----------|-------|
| [TECHNICAL_DESIGN_DATA.md](./TECHNICAL_DESIGN_DATA.md) | Database schemas, entity relationships, ID generation, cascade deletes |
| [TECHNICAL_DESIGN_RUNTIME.md](./TECHNICAL_DESIGN_RUNTIME.md) | Runner logic, queue mechanics, CLI adapters, context passing, background jobs |
| [TECHNICAL_DESIGN_API.md](./TECHNICAL_DESIGN_API.md) | REST endpoints, SSE events, real-time updates |
| [TECHNICAL_DESIGN_UX.md](./TECHNICAL_DESIGN_UX.md) | User flows, detailed interactions, ASCII mockups |
| [TECHNICAL_DESIGN_UI.md](./TECHNICAL_DESIGN_UI.md) | React components, state management, build/distribution |

---

## Key Architectural Decisions

### ADR-001: Single Binary Distribution

**Decision:** Malamar is distributed as a single executable binary with embedded UI assets.

**Context:** Target users are developers who want quick setup without dependency management.

**Rationale:**
- Zero external dependencies (no Redis, no external database, no Node.js runtime)
- Simple installation: download and run, or `bunx`/`npx`
- Pocketbase-like simplicity is a key differentiator
- Homebrew support for easy updates

**Implications:**
- SQLite instead of PostgreSQL/MySQL
- In-process event bus instead of Redis pub/sub
- UI assets bundled into binary or extracted on startup

---

### ADR-002: SQLite as Database

**Decision:** Use SQLite for all data persistence.

**Context:** Need a database that supports the single-binary distribution model.

**Rationale:**
- No separate database server to install or manage
- File-based storage in `~/.malamar/malamar.db`
- Sufficient for single-user, local-first use case
- Backup is just copying a file

**Implications:**
- No horizontal scaling (acceptable for target use case)
- Write concurrency limited (mitigated by queue-based processing)
- Must handle migrations carefully on startup

---

### ADR-003: Event Queue Pattern for Task Processing

**Decision:** Use an event queue to manage task processing rather than direct execution.

**Context:** Tasks trigger agent loops that may take a long time and need to be resumable.

**Rationale:**
- Decouples task events from processing
- Natural retry mechanism (new queue item on error)
- Enables priority override without interrupting running tasks
- Survives restarts (queue items persisted in database)
- LIFO ordering lets users "bump" tasks by commenting

**Implications:**
- Slight delay between event and processing (polling interval)
- Queue cleanup job needed to prevent unbounded growth
- Must handle "in_progress" items on startup

---

### ADR-004: CLI Subprocess Model

**Decision:** Invoke AI CLIs as subprocesses rather than using APIs directly.

**Context:** Users already have CLI tools installed and configured with their own API keys.

**Rationale:**
- Leverage existing CLI installations and configurations
- No need to manage API keys within Malamar
- Each CLI handles its own authentication, rate limiting, etc.
- Uniform interface via file-based context passing
- Future-proof for new CLIs (just add adapter)

**Implications:**
- Dependent on CLI availability and health
- Must handle subprocess failures gracefully
- Context passing via temp files (not in-memory)
- Response format must be enforced via prompts or CLI flags

---

### ADR-005: File-Based Context Passing

**Decision:** Pass context to CLIs via markdown input files and receive responses via JSON output files.

**Context:** CLIs need full task context but have varying interfaces.

**Rationale:**
- Works with any CLI that can read files
- Markdown is human-readable for debugging
- JSONL format for comments/logs prevents parsing issues
- Random output filenames prevent stale data bugs
- Files serve as audit trail during debugging

**Implications:**
- Temp file management (rely on OS cleanup)
- Must pre-create output files to ensure path exists
- File size could be large for tasks with many comments

---

### ADR-006: One Task Per Workspace Concurrency

**Decision:** Only one task is processed at a time per workspace.

**Context:** Agents may work on shared resources (files, git repo) within a workspace.

**Rationale:**
- Prevents conflicts when multiple agents modify the same files
- Simplifies working directory management
- Predictable resource usage
- Auto-demotion keeps UI clear (only one "In Progress")

**Implications:**
- Tasks queue up within a workspace
- Priority override needed for urgent tasks
- Multiple workspaces can process in parallel

---

### ADR-007: Chat vs Task Separation

**Decision:** Chat is a separate feature from tasks, not integrated into task comments.

**Context:** Users need both autonomous task processing and interactive conversations.

**Rationale:**
- Different mental models: tasks are autonomous, chats are interactive
- Chats can run concurrently (unlike tasks)
- Chat with Malamar agent enables workspace management
- Agents in chat can create tasks (bridging the two)

**Implications:**
- Separate queue for chat processing
- Different context file formats
- Chat actions (create_task, create_agent) vs task actions (skip, comment, change_status)

---

## Cross-Cutting Concerns

### Error Handling Philosophy

**Principle:** Errors should be visible and self-healing when possible.

| Layer | Error Handling |
|-------|----------------|
| **CLI Execution** | System comment with details → retry via queue → agents can self-correct |
| **Malformed Output** | Treated same as CLI failure → visible error, automatic retry |
| **Chat Processing** | System message in chat → user sees immediately, can retry |
| **API Requests** | Standard HTTP error codes → client handles retry/display |
| **Background Jobs** | Logged, continue processing other items |
| **Notifications** | Silent failure (logged) → non-critical path |

**No Silent Failures:** Errors in task processing always result in a visible System comment. Users are never left wondering why nothing happened.

**Contextual Retry:** By writing error details to comments, subsequent agent loops have context to potentially fix the issue themselves.

---

### Security Model

**Current State:** No authentication. Malamar is designed for local-only or Tailscale-protected access.

**Trust Boundaries:**
- Web UI ↔ API: Trusted (same process, no auth)
- API ↔ Database: Trusted (local SQLite file)
- Malamar ↔ CLIs: Trusted (user's installed tools)
- CLIs ↔ External APIs: CLI's responsibility

**Sensitive Data:**
- API keys stored in CLI settings (user's responsibility to secure `~/.malamar`)
- Mailgun credentials in settings table
- No encryption at rest (SQLite file)

**Future Considerations:**
- Multi-user authentication (workspace-scoped SSE, access control)
- Encrypted settings storage
- Audit logging for sensitive operations

---

### Observability

**Logging:**
- Configurable log level: `debug`, `info`, `warn`, `error`
- Configurable format: `text` (human), `json` (machine)
- No external log aggregation (local stdout/stderr)

**Metrics:**
- No built-in metrics endpoint (future consideration)
- CLI health status available via API

**Tracing:**
- Activity logs provide task-level audit trail
- System comments capture error context
- Temp files can be inspected for debugging

---

### Temp File Management

**Philosophy:** Malamar creates temp files but does not proactively clean them up.

**Locations:**
- Input/output files: `/tmp/malamar_*`
- Task working directories: `/tmp/malamar_tasks_{task_id}`
- Chat working directories: `/tmp/malamar_chat_{chat_id}`
- Chat attachments: `/tmp/malamar_chat_{chat_id}_attachments/`

**Cleanup Strategy:**
- Rely on OS `/tmp` cleanup (acceptable for most users)
- Users needing control can set `MALAMAR_TEMP_DIR` to manage their own directory

**Rationale:** Keeping temp files aids debugging. Aggressive cleanup could hide issues.

---

### Deployment Model

**Local Development:**
```
./malamar
# or
bun run dev
```

**Production (Local):**
```
# Download binary
./malamar

# Or via package manager
brew install malamar
malamar
```

**Remote Access:**
- Tailscale recommended for secure remote access
- No built-in HTTPS (reverse proxy if needed)
- No authentication (Tailscale provides identity)

**Data Location:**
- Database: `~/.malamar/malamar.db`
- UI assets: embedded or `~/.malamar/ui/`
- Temp files: `/tmp` or `MALAMAR_TEMP_DIR`

---

### Extensibility Points

**CLI Adapters:**
- Hardcoded adapter per supported CLI
- Adding new CLI: create adapter with command template + response format enforcement
- User customization limited to binary path and environment variables

**Agent Instructions:**
- All domain-specific behavior defined in agent instructions
- Malamar is domain-agnostic by design
- Users can create any workflow by writing instructions

**Malamar Agent:**
- Instruction hardcoded in binary
- Knowledge base fetched from remote URL (can evolve independently)
- Actions are predefined (create/update/delete agents, etc.)

**Future Extensibility:**
- Plugin system for CLI adapters (not planned for v1)
- Custom action types (not planned for v1)
- Webhook integrations (not planned for v1)
