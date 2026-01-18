# Malamar

> **Malamar lets you combine the strengths of different AI CLIs into autonomous multi-agent workflows, limited only by your creativity.**

## What is Malamar?

Malamar is a self-contained orchestration layer that coordinates multiple AI CLI tools (Claude Code, Gemini CLI, Codex CLI, OpenCode) into automated multi-agent workflows. Instead of manually switching between tools, you define agents with specific roles (Planner, Implementer, Reviewer, Approver), assign them to your preferred CLIs, and let Malamar run them autonomously until your task is complete.

## Documentation

### Product Specifications (What & Why)

- **[SPECS.md](./SPECS.md)** - Product specifications: problem statement, core concepts, entities, behaviors, and UI overview.

### Technical Design (How)

- **[TECHNICAL_DESIGN.md](./TECHNICAL_DESIGN.md)** - Architectural overview, system diagrams, key design decisions, cross-cutting concerns.
  - [TECHNICAL_DESIGN_DATA.md](./TECHNICAL_DESIGN_DATA.md) - Database schemas, entity relationships, ID generation, cascade deletes.
  - [TECHNICAL_DESIGN_RUNTIME.md](./TECHNICAL_DESIGN_RUNTIME.md) - Runner logic, queue mechanics, CLI adapters, context passing.
  - [TECHNICAL_DESIGN_API.md](./TECHNICAL_DESIGN_API.md) - REST endpoints, SSE events, real-time updates.
  - [TECHNICAL_DESIGN_UX.md](./TECHNICAL_DESIGN_UX.md) - User flows, detailed interactions, ASCII mockups, UI/UX decisions.
  - [TECHNICAL_DESIGN_UI.md](./TECHNICAL_DESIGN_UI.md) - React components, state management, implementation patterns.

### Knowledge Base

- **[AGENTS/](./AGENTS/)** - Knowledge base for the Malamar agent: concepts, patterns, and troubleshooting guidance the AI accesses remotely (via GitHub URL) when answering user messages.

---

## Tech Stack

**Backend:**
- TypeScript with Bun runtime
- RESTful API
- Background jobs (runner, cleanup, health check)
- SSE endpoint for real-time updates
- SQLite database

**Frontend:**
- TypeScript with React (via create-vite-app)
- Bun as the build tool/runtime

**Distribution:**
- Single executable binary, or run via `bunx`/`npx`
- Can be installed with Homebrew

---

## Configuration

Configuration via environment variables (priority) or CLI flags.

### Server

| Setting | Env Var | CLI Flag | Default |
|---------|---------|----------|---------|
| Bind address | `MALAMAR_HOST` | `--host` | `127.0.0.1` |
| Server port | `MALAMAR_PORT` | `--port` | `3456` |

### Data & Storage

| Setting | Env Var | CLI Flag | Default |
|---------|---------|----------|---------|
| Data directory | `MALAMAR_DATA_DIR` | `--data-dir` | `~/.malamar` |

### Logging

| Setting | Env Var | CLI Flag | Default |
|---------|---------|----------|---------|
| Log verbosity | `MALAMAR_LOG_LEVEL` | `--log-level` | `info` |
| Output format | `MALAMAR_LOG_FORMAT` | `--log-format` | `text` |

Log levels: `debug`, `info`, `warn`, `error`

Log formats: `text`, `json`

### Runner

| Setting | Env Var | CLI Flag | Default |
|---------|---------|----------|---------|
| Poll interval (ms) | `MALAMAR_RUNNER_POLL_INTERVAL` | `--runner-poll-interval` | `1000` |

### Temp Files

| Setting | Env Var | CLI Flag | Default |
|---------|---------|----------|---------|
| Temp directory | `MALAMAR_TEMP_DIR` | `--temp-dir` | System `/tmp` |

**Cleanup Policy:** Malamar does not proactively clean up its own temp files. OS-dependent `/tmp` cleanup is acceptable. Users needing control can use `MALAMAR_TEMP_DIR` to manage their own temp directory.

---

## CLI Commands

| Command | Description |
|---------|-------------|
| `malamar` | Start the server (default) |
| `malamar version` | Show version info |
| `malamar help` | Show help/usage |
| `malamar doctor` | Check system health (CLIs, database, config) |
| `malamar config` | Show current configuration |
| `malamar export` | Export data (details TBD) |
| `malamar import` | Import data (details TBD) |