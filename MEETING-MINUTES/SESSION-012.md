# Session 012

Please read [SKILL.md](../.claude/skills/explore-specs/SKILL.md) to understand the process used to create this file.

**Session Focus:** Backend repository structure - folder organization, module patterns, file conventions, testing strategy, and development tooling.

---

## Question #1: Backend Repository Structure Philosophy

Before diving into specifics, the guiding principle for the backend codebase organization was established. Given Malamar's characteristics (single-process app, TypeScript/Bun, Hono, SQLite, domain concepts like workspaces/agents/tasks/chats), what architectural approach should the backend follow?

### Answer

**Modular monolith ("monolith module") structure**: Each domain is a self-contained module while sharing core infrastructure.

```
src/
  core/
  workspace/
  task/
  instructions/
  ...
```

This approach scales well and keeps related code together. Domains are organized as modules (workspace, task, chat, etc.) with a `core/` folder for shared infrastructure.

---

## Question #2: Module Internal Structure

For a domain module like `workspace/`, what internal structure should it follow?

### Answer

**Flat with conventions**: Single level of files with naming conventions, no sub-folders.

```
src/workspace/
├── index.ts         # Public exports (routes, types)
├── routes.ts        # Hono route handlers
├── service.ts       # Business logic  
├── repository.ts    # Database queries
├── schemas.ts       # Zod request/response schemas
└── types.ts         # TypeScript types/interfaces
```

For a codebase of this size, sub-folders add navigation overhead without benefit. Flat structure with clear file names is easier to scan. The `index.ts` provides a clean public API for each module.

---

## Question #3: The `core/` Module Contents

What should live in the `core/` module?

### Answer

**Infrastructure-only**: Only technical infrastructure that has no domain knowledge.

```
src/core/
├── index.ts         # Public exports
├── database.ts      # SQLite connection, PRAGMA setup, transaction helpers
├── events.ts        # EventEmitter singleton for SSE pub/sub
├── logger.ts        # Logger with level/format support
├── config.ts        # Configuration loading (env vars, CLI flags, defaults)
├── errors.ts        # Error classes (NotFoundError, ValidationError, etc.)
└── types.ts         # Shared utility types
```

Clear boundary: if it knows about workspaces, tasks, or agents, it doesn't belong in core. This keeps core stable and domain modules independent.

---

## Question #4: Shared Utilities Placement

With `core/` being infrastructure-only, where should shared utilities and cross-cutting concerns live?

### Answer

**Place utilities in relevant domain modules + create `shared/` module**: Utilities live where they're primarily used, export if needed elsewhere. Additionally, a `shared/` module exists for cross-cutting types and utilities that don't have a clear domain home.

Examples:
- `src/core/nanoid.ts` - ID generation (truly generic)
- `src/core/datetime.ts` - Date formatting (truly generic)
- `src/runner/subprocess.ts` - Subprocess tracking (runner owns this)
- `src/events/sse.ts` - SSE registry (events module)
- `src/notifications/mailgun.ts` - Email (notifications module)
- `src/shared/types.ts` - Cross-cutting types used by multiple modules

---

## Question #5: Background Jobs Placement

Where should background jobs (Runner, Cleanup, Health Check) live in the structure?

### Answer

**Runner as its own module, simpler jobs in `jobs/`**: The runner is complex enough to warrant its own module.

```
src/runner/
├── index.ts         # Runner loop entry point
├── task-worker.ts   # Task processing logic
├── chat-worker.ts   # Chat processing logic
├── subprocess.ts    # Subprocess tracking
└── types.ts

src/jobs/
├── index.ts         # Job registration
├── cleanup.ts
└── health-check.ts
```

The runner is the heart of Malamar - it manages the multi-agent loop, subprocess tracking, queue pickup algorithm, and error handling. It deserves first-class module status. Cleanup and health-check are simpler scheduled tasks.

---

## Question #6: CLI Adapters Placement

Where should CLI adapter logic live?

### Answer

**Dedicated `cli/` module**: All CLI-related logic in one place.

```
src/cli/
├── index.ts         # Adapter factory, CLI type exports
├── types.ts         # CliType enum, adapter interface
├── adapters/
│   ├── claude.ts
│   ├── gemini.ts
│   ├── codex.ts
│   └── opencode.ts
└── health.ts        # Health check logic (used by jobs/health-check.ts)
```

CLI adapters are a cohesive unit with clear boundaries. The runner imports and uses them, the health-check job calls their health methods.

---

## Question #7: API Routes Organization

How should routes be organized and mounted?

### Answer

**Routes in domain modules, central app assembly**: Each module defines its routes, main app imports and mounts them.

```typescript
// src/workspace/routes.ts
export const workspaceRoutes = new Hono()...

// src/app.ts - Central assembly
import { workspaceRoutes } from './workspace';
import { taskRoutes } from './task';

app.route('/api/workspaces', workspaceRoutes);
app.route('/api/tasks', taskRoutes);
```

Each domain owns its routes. The central `app.ts` provides a clear map of all API endpoints, making it easy to see the full API surface in one place.

---

## Question #8: Entry Point and Startup Organization

How should startup logic be organized?

### Answer

**Single `index.ts` entry point**: One file orchestrates startup, delegates to modules.

```
src/index.ts          # Entry point
├── Load config from core/config
├── Initialize database from core/database
├── Run migrations
├── Check first startup, create sample workspace
├── Start background jobs
├── Create app from app.ts
├── Start server
└── Register shutdown handlers
```

Clear single entry point. Each step delegates to the relevant module. Easy to understand the startup sequence by reading one file.

---

## Question #9: Database Migrations Location

Where should migration files live?

### Answer

**Top-level `migrations/` folder**: Separate from source code.

```
migrations/
├── 001_initial_schema.sql
├── 002_add_chat_tables.sql
└── ...

src/
└── ...
```

Migrations are data, not code. Keeping them at the top level makes them easy to find and manage. Common convention in many frameworks (Rails, Django, Prisma). The migration runner in `core/database.ts` references this folder.

---

## Question #10: SSE Events Module

Where should SSE logic live?

### Answer

**Dedicated `events/` module**: SSE is significant enough for its own module.

```
src/events/
├── index.ts         # Public exports
├── emitter.ts       # EventEmitter singleton
├── registry.ts      # Connection registry (Set<Response>)
├── broadcast.ts     # Broadcasting logic
├── routes.ts        # GET /api/events endpoint
└── types.ts         # Event type definitions
```

SSE is a distinct subsystem with its own concerns. Domain modules import `emit()` from events to broadcast. The events module owns the full SSE lifecycle.

---

## Question #11: Notifications Module

Where should notification logic live?

### Answer

**Dedicated `notifications/` module**: Own module for notification concerns.

```
src/notifications/
├── index.ts         # Public exports (notify function)
├── mailgun.ts       # Mailgun API client
├── service.ts       # Check settings, send if enabled
└── types.ts         # Notification event types
```

Notifications are a distinct feature. The runner imports and calls `notify()`. The test-email endpoint lives in `settings/routes.ts` (it's a settings action) and calls the notifications module.

---

## Question #12: Domain Module List

The complete list of modules for the backend.

### Answer

**Core/Infrastructure:**
- `core/` - Database, logger, config, errors
- `shared/` - Cross-cutting types and utilities
- `events/` - SSE subsystem

**Domain Modules:**
- `workspace/` - Workspace CRUD, settings
- `agent/` - Agent CRUD, reordering
- `task/` - Task CRUD, comments, activity logs, prioritization
- `chat/` - Chat CRUD, messages, attachments
- `cli/` - CLI adapters with individual adapter files
- `settings/` - Global settings (Mailgun config, CLI settings)
- `notifications/` - Email notifications
- `instructions/` - Default prompts, onboarding instructions
- `health/` - Health check endpoints (`/api/health`, `/api/health/cli`)

**Jobs/Runner:**
- `runner/` - Task/chat queue processing, subprocess management
- `jobs/` - Cleanup, health-check scheduled jobs

**Commands:**
- `commands/` - CLI commands (serve, version, help, doctor, config)

**Application:**
- `app.ts` - Hono app assembly
- `index.ts` - Entry point

---

## Question #13: CLI Commands Structure

How should CLI command handling be organized?

### Answer

**Dedicated `commands/` module**: Each command in its own file for consistency and reduced cognitive load.

```
src/commands/
├── index.ts         # Parse args, dispatch to command
├── serve.ts         # Default command (start server)
├── version.ts       # Print version
├── help.ts          # Print usage
├── doctor.ts        # System health check
└── config.ts        # Print current config

src/index.ts         # Minimal: import commands, run
```

Consistent with the modular approach. Predictable and easy to navigate.

---

## Question #14: Static Frontend Files Location

Where should the frontend build output be referenced during development?

### Answer

The frontend/backend repo structure is TBD (could be monorepo or separate repos), but for the backend side:

```
(backend repo or package)
├── src/
├── migrations/
└── public/          # Frontend build output copied here before BE build
```

A build script copies frontend assets into `public/` before compiling the backend binary. This keeps the backend structure clean.

---

## Question #15: Test Files Location

Where should test files live?

### Answer

**Three test types with distinct locations:**

| Type | Location | What it tests | Server running? |
|------|----------|---------------|-----------------|
| **Unit** | `src/**/*.test.ts` | Individual functions, mocked dependencies | No |
| **Integration** | `tests/` | Function calls, real DB, service layers | No |
| **E2E** | `e2e/` | HTTP requests, assert response + query DB/files | Yes |

Unit tests are co-located with source files for discoverability. Integration and E2E tests have separate directories.

---

## Question #16: TypeScript Configuration and Build Output

How should TypeScript compilation and build artifacts be organized?

### Answer

**No separate build output**: Bun runs TypeScript directly, `bun build --compile` produces the binary.

```
src/               # TypeScript source
migrations/
public/

# No dist/ folder - Bun runs .ts directly in dev
# bun build --compile --outfile malamar produces the binary
```

Bun runs TypeScript natively without a transpilation step in development. For production, `bun build --compile` creates a self-contained binary. No intermediate `dist/` folder needed.

---

## Question #17: File Naming Conventions

What naming conventions should be used for files?

### Answer

**kebab-case for all files**:

```
src/workspace/
├── index.ts
├── routes.ts
├── service.ts
└── types.ts

src/runner/
├── task-worker.ts      # Multi-word: kebab-case
└── chat-worker.ts
```

Consistent with npm ecosystem conventions. Works across all operating systems. Easy to type.

---

## Question #18: Import Path Conventions

How should imports be organized within the codebase?

### Answer

**Relative imports throughout, with `eslint-plugin-simple-import-sort` for automatic sorting**:

```typescript
// Inside src/workspace/service.ts
import { WorkspaceRow } from './types';           // Same module: relative
import { db } from '../core/database';            // Cross-module: relative from src
import { emit } from '../events';                 // Cross-module: relative from src
```

No tsconfig path aliases to configure. Works out of the box with Bun. The `eslint-plugin-simple-import-sort` plugin (https://github.com/lydell/eslint-plugin-simple-import-sort) handles automatic import organization via `eslint --fix`.

---

## Question #19: Complete Backend Structure Summary

The complete backend repository structure.

### Answer

```
malamar-backend/
├── src/
│   ├── index.ts                    # Entry point, orchestrates startup
│   ├── app.ts                      # Hono app assembly, mounts all routes
│   │
│   ├── commands/                   # CLI commands
│   │   ├── index.ts
│   │   ├── serve.ts
│   │   ├── version.ts
│   │   ├── help.ts
│   │   ├── doctor.ts
│   │   └── config.ts
│   │
│   ├── core/                       # Infrastructure (no domain knowledge)
│   │   ├── index.ts
│   │   ├── database.ts
│   │   ├── config.ts
│   │   ├── logger.ts
│   │   ├── errors.ts
│   │   └── types.ts
│   │
│   ├── shared/                     # Cross-cutting types/utilities
│   │   ├── index.ts
│   │   ├── nanoid.ts
│   │   ├── datetime.ts
│   │   └── types.ts
│   │
│   ├── workspace/                  # Domain module (flat structure)
│   │   ├── index.ts
│   │   ├── routes.ts
│   │   ├── service.ts
│   │   ├── repository.ts
│   │   ├── schemas.ts
│   │   ├── types.ts
│   │   └── service.test.ts         # Unit test (co-located)
│   │
│   ├── agent/                      # Same flat structure
│   ├── task/
│   ├── chat/
│   ├── settings/
│   ├── health/
│   │
│   ├── cli/                        # CLI adapters
│   │   ├── index.ts
│   │   ├── types.ts
│   │   ├── health.ts
│   │   └── adapters/
│   │       ├── claude.ts
│   │       ├── gemini.ts
│   │       ├── codex.ts
│   │       └── opencode.ts
│   │
│   ├── runner/                     # Task/chat queue processing
│   │   ├── index.ts
│   │   ├── task-worker.ts
│   │   ├── chat-worker.ts
│   │   ├── subprocess.ts
│   │   └── types.ts
│   │
│   ├── jobs/                       # Scheduled jobs
│   │   ├── index.ts
│   │   ├── cleanup.ts
│   │   └── health-check.ts
│   │
│   ├── events/                     # SSE subsystem
│   │   ├── index.ts
│   │   ├── emitter.ts
│   │   ├── registry.ts
│   │   ├── broadcast.ts
│   │   ├── routes.ts
│   │   └── types.ts
│   │
│   ├── notifications/              # Email notifications
│   │   ├── index.ts
│   │   ├── mailgun.ts
│   │   ├── service.ts
│   │   └── types.ts
│   │
│   └── instructions/               # Default prompts, onboarding
│       ├── index.ts
│       └── sample-workspace.ts
│
├── migrations/                     # SQL migration files
│   ├── 001_initial_schema.sql
│   └── ...
│
├── public/                         # Frontend build (populated before BE build)
│
├── tests/                          # Integration tests
│   ├── helpers/
│   │   └── db.ts
│   ├── workspace/
│   │   ├── service.test.ts
│   │   └── repository.test.ts
│   └── ...
│
├── e2e/                            # E2E tests
│   ├── helpers/
│   │   └── server.ts
│   ├── workspace.test.ts
│   ├── task.test.ts
│   └── chat.test.ts
│
├── package.json
├── tsconfig.json
├── .eslintrc.js
├── .prettierrc
└── bunfig.toml
```

**Conventions:**
- File naming: kebab-case
- Imports: Relative paths, no aliases
- Unit tests: Co-located (`*.test.ts`)
- Integration tests: `tests/`
- E2E tests: `e2e/`

---

## Question #20: Development Tooling Summary

What development tooling should the backend use?

### Answer

| Tool | Purpose |
|------|---------|
| **Bun** | Runtime, package manager, test runner, bundler |
| **TypeScript** | Type safety (run directly via Bun) |
| **Hono** | Web framework |
| **Zod** | Request validation + type inference |
| **Prettier** (https://prettier.io/) | Code formatting |
| **ESLint** | Linting |
| **eslint-plugin-simple-import-sort** | Import/export sorting |

---

## Question #21: AI Agent Development Environment Design

How should the development environment be designed to allow AI agents to perform E2E testing workflows (clean data, start instance, send requests, query DB)?

### Answer

**CLI commands + standard tools approach**: Use standard tools AI agents already know (curl, sqlite3, shell commands). The E2E test suite handles the full lifecycle.

Key mechanisms:
- Use `MALAMAR_DATA_DIR` environment variable for isolated test directories
- Use `MALAMAR_PORT` for custom ports
- Health endpoint for readiness checking
- Direct SQLite access for assertions

---

## Question #22: E2E Test Suite Structure

How should the E2E test suite handle server lifecycle?

### Answer

**Self-contained test files with per-file server lifecycle**: Each E2E test file manages its own server startup and shutdown via `beforeAll`/`afterAll`.

This enables running a single test file: `bun test e2e/workspace.test.ts`

---

## Question #23: Self-Contained E2E Test Pattern

The detailed pattern for self-contained E2E tests.

### Answer

```typescript
// e2e/helpers/server.ts
const TEST_PORT = 3457;
const TEST_DATA_DIR = '/tmp/malamar-test';

export async function startServer() {
  // Clean before (guarantee clean slate)
  await Bun.$`rm -rf ${TEST_DATA_DIR}`.quiet();
  
  // Start server with test config
  server = spawn({
    cmd: ['bun', 'run', 'src/index.ts'],
    env: { 
      ...process.env,
      MALAMAR_PORT: String(TEST_PORT),
      MALAMAR_DATA_DIR: TEST_DATA_DIR,
    },
  });

  // Wait for healthy
  await waitForHealthy(`http://localhost:${TEST_PORT}/api/health`);
}

export async function stopServer() {
  server.kill();
  // Clean after
  await Bun.$`rm -rf ${TEST_DATA_DIR}`.quiet();
}

export function getDb() {
  return new Database(`${TEST_DATA_DIR}/malamar.db`, { readonly: true });
}
```

```typescript
// e2e/workspace.test.ts
describe('Workspaces E2E', () => {
  beforeAll(async () => { await startServer(); });
  afterAll(async () => { await stopServer(); });

  test('create workspace and verify in DB', async () => {
    const res = await fetch(`${getBaseUrl()}/api/workspaces`, {
      method: 'POST',
      body: JSON.stringify({ title: 'Test' }),
    });
    expect(res.status).toBe(201);

    // Verify in database
    const row = getDb().query('SELECT * FROM workspaces WHERE id = ?').get(id);
    expect(row.title).toBe('Test');
  });
});
```

**Key features:**
- Fixed test data directory: `/tmp/malamar-test`
- Cleanup before and after tests
- Never touches `~/.malamar`
- Direct DB access for assertions

---

## Question #24: Updated Test Directory Structure

The complete test structure with three test types.

### Answer

```
src/
├── workspace/
│   ├── service.ts
│   └── service.test.ts       # Unit test (co-located)
└── ...

tests/                         # Integration tests
├── helpers/
│   └── db.ts                 # Test DB setup
├── workspace/
│   ├── service.test.ts       # Test service with real DB, no server
│   └── repository.test.ts
└── ...

e2e/                           # E2E tests
├── helpers/
│   └── server.ts             # startServer, stopServer, getBaseUrl, getDb
├── workspace.test.ts
├── task.test.ts
└── chat.test.ts

public/
migrations/
```

**Test commands:**
```bash
bun test src/              # Unit tests only
bun test tests/            # Integration tests only
bun test e2e/              # E2E tests only
bun test e2e/workspace.test.ts  # Single E2E file
```
