# Session 013

Please read [SKILL.md](../.claude/skills/explore-specs/SKILL.md) to understand the process used to create this file.

**Session Focus:** Frontend development enablement - folder organization, component patterns, data fetching, state management, styling conventions, and development tooling needed to start frontend development.

**Tech Stack:** Vite, TypeScript, Bun, React Router, React Query, Zustand, shadcn/ui, Prettier, ESLint

---

## Question #1: Frontend Repository Structure Philosophy

Before diving into specifics, the guiding principle for the frontend codebase organization was established. Given Malamar's characteristics (React + TypeScript, shadcn/ui + Tailwind, mobile-first SPA with Kanban boards, chat interfaces, real-time updates via SSE), what folder organization approach should the frontend follow?

### Answer

**Feature-based organization**: Organize by domain feature (workspace, task, chat, settings), with each feature containing its own components, hooks, and utilities. Shared infrastructure lives in dedicated folders.

```
src/
  features/
    workspace/
    task/
    chat/
    settings/
  components/      # Shared UI components
  hooks/           # Shared hooks
  lib/             # Utilities
```

This approach scales well, keeps related code together, and aligns with the backend's domain module structure from SESSION-012.

---

## Question #2: Feature Module Internal Structure

For a feature module like `workspace/`, what internal structure should it follow?

### Answer

**Nested sub-folders with pages**: Each feature has sub-folders for organization.

```
src/features/workspace/
├── index.ts              # Public exports
├── pages/
│   ├── WorkspaceList.tsx
│   └── WorkspaceDetail.tsx
├── components/
│   ├── WorkspaceCard.tsx
│   └── WorkspaceForm.tsx
├── hooks/
│   └── use-workspaces.ts
├── api/
│   └── workspace.api.ts
└── types/
    └── workspace.types.ts
```

---

## Question #3: Shared Infrastructure Folders

With feature modules containing their own `pages/`, `components/`, `hooks/`, and `api/`, what shared folders should exist at the `src/` level for cross-cutting concerns?

### Answer

**Comprehensive shared structure**:

```
src/
├── features/           # Feature modules (workspace, task, chat, settings)
├── components/         # shadcn/ui components + shared UI primitives
│   └── ui/            # shadcn/ui generated components
├── hooks/              # Shared hooks (useSSE, useToast, useTheme)
├── lib/                # Utilities (api-client, date formatting, cn helper)
├── stores/             # Zustand stores (when needed)
├── routes/             # React Router configuration
├── types/              # Shared TypeScript types
└── App.tsx             # Root component
```

Based on common Vite + React patterns and shadcn/ui's default structure (which uses `components/ui/`).

---

## Question #4: API Client Layer Design

With React Query for data fetching, how should the API client layer be structured?

### Answer

**Fetch wrapper + feature API files**:

- A thin `api-client.ts` in `lib/` that wraps `fetch` with base URL, error handling, and JSON parsing
- Each feature has an `api/` folder with functions like `getWorkspaces()`, `createWorkspace(data)`
- React Query hooks in `hooks/` call these API functions

```typescript
// lib/api-client.ts
export const apiClient = {
  get: <T>(path: string) => fetch(`/api${path}`).then(handleResponse<T>),
  post: <T>(path: string, data: unknown) => fetch(...).then(handleResponse<T>),
  // ...
};

// features/workspace/api/workspace.api.ts
export const workspaceApi = {
  list: (query?: string) => apiClient.get<Workspace[]>(`/workspaces${query ? `?q=${query}` : ''}`),
  get: (id: string) => apiClient.get<Workspace>(`/workspaces/${id}`),
  create: (data: CreateWorkspaceInput) => apiClient.post<Workspace>('/workspaces', data),
};

// features/workspace/hooks/use-workspaces.ts
export const useWorkspaces = (query?: string) => {
  return useQuery({ queryKey: ['workspaces', query], queryFn: () => workspaceApi.list(query) });
};
```

---

## Question #5: React Query Configuration and Patterns

React Query needs configuration for defaults (stale time, retry behavior, error handling) and patterns for mutations. Given Malamar's real-time nature (SSE updates, 3-second polling on task detail), how should React Query be configured?

### Answer

**Conservative defaults with SSE-driven invalidation**:

- `staleTime: 30000` (30s) - data stays fresh longer since SSE pushes updates
- `gcTime: 300000` (5min) - keep unused data in cache
- `refetchOnWindowFocus: true` - refresh when user returns
- `retry: 1` - single retry on failure (errors surface quickly)
- SSE events trigger `queryClient.invalidateQueries()` for affected resources
- Task detail page: override with `refetchInterval: 3000` as specified in TECHNICAL_DESIGN.md

```typescript
// lib/query-client.ts
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30 * 1000,
      gcTime: 5 * 60 * 1000,
      refetchOnWindowFocus: true,
      retry: 1,
    },
  },
});
```

---

## Question #6: SSE Subscription and Event Handling

The backend exposes `GET /api/events` for Server-Sent Events. How should SSE be implemented on the frontend?

### Answer

**Centralized SSE hook + event bus pattern**:

- Single `useSSE` hook in `hooks/` manages the EventSource connection
- Zustand store tracks connection state (connected, reconnecting, error)
- Events dispatched to handlers registered by features
- Auto-reconnect with exponential backoff (SSE `retry:` hint from server as baseline)
- Query invalidation and toast notifications as event handlers

```typescript
// hooks/use-sse.ts - initializes connection, manages lifecycle
// stores/sse-store.ts - connection state
// lib/sse-handlers.ts - maps event types to handlers

// In App.tsx or layout:
useSSE(); // Establishes connection once

// Handler example:
sseHandlers.register('task.status_changed', (data) => {
  queryClient.invalidateQueries({ queryKey: ['tasks', data.workspace_id] });
  queryClient.invalidateQueries({ queryKey: ['task', data.task_id] });
  showToast({ title: `Task moved to ${data.new_status}`, ... });
});
```

---

## Question #7: Zustand Store Organization

With Zustand confirmed for state management, what global state needs to be managed outside of React Query?

### Answer

**Use Zustand when needed**: React Query handles server-state (workspaces, tasks, chats). Zustand should handle client-only state. No stores are pre-defined - create them as use cases arise. The SSE connection state is a likely candidate when implementation starts.

---

## Question #8: Routing Structure with React Router

How should routes be organized?

### Answer

**Flat route config with lazy loading**:

- Single `routes/index.tsx` defines all routes with `lazy()` for code splitting
- Routes mirror the page hierarchy from TECHNICAL_DESIGN.md

```typescript
// routes/index.tsx
export const router = createBrowserRouter([
  {
    path: '/',
    element: <RootLayout />,
    children: [
      { index: true, lazy: () => import('@/features/workspace/pages/WorkspaceList') },
      { path: 'workspaces/:workspaceId', lazy: () => import('@/features/workspace/pages/WorkspaceDetail') },
      { path: 'workspaces/:workspaceId/tasks/:taskId', lazy: () => import('@/features/task/pages/TaskDetail') },
      { path: 'workspaces/:workspaceId/chats/:chatId', lazy: () => import('@/features/chat/pages/ChatDetail') },
      { path: 'settings', lazy: () => import('@/features/settings/pages/GlobalSettings') },
    ],
  },
]);
```

---

## Question #9: Task and Chat Detail View Pattern

Should Task Detail and Chat Detail be modal overlays or full separate pages?

### Answer

**Modal overlays with URL state**:

- Task/Chat detail renders as a modal over the Workspace Detail page
- URL updates to `/workspaces/:id/tasks/:taskId` (enables deep linking, back button works)
- On mobile: modal becomes full-screen (still a modal, just larger)
- Closing modal navigates back to `/workspaces/:id`
- React Router's `<Outlet>` in Workspace Detail renders the modal when child route matches

```typescript
// WorkspaceDetail.tsx
return (
  <div>
    <Tabs>...</Tabs>
    <KanbanBoard />
    <Outlet /> {/* Renders TaskDetail modal when route matches */}
  </div>
);
```

---

## Question #10: Form Handling Approach

How should form state and validation be handled?

### Answer

**React Hook Form + Zod**:

- `react-hook-form` for form state management (performant, minimal re-renders)
- `@hookform/resolvers/zod` for Zod schema validation
- Schemas defined per-feature in `types/` or alongside components
- shadcn/ui `<Form>` components integrate with React Hook Form

```typescript
// features/workspace/components/WorkspaceForm.tsx
const schema = z.object({
  title: z.string().min(1, 'Title is required'),
  description: z.string().optional(),
});

const form = useForm<z.infer<typeof schema>>({
  resolver: zodResolver(schema),
  defaultValues: { title: '', description: '' },
});
```

---

## Question #11: Component Patterns and Conventions

How should feature-specific and shared composite components be organized and named?

### Answer

**Composition-based with clear naming**:

- `components/ui/` - shadcn/ui primitives (Button, Card, Dialog, etc.) - don't modify these
- `components/` - Shared composite components built from ui primitives (e.g., `ConfirmDialog.tsx`, `SearchInput.tsx`, `EmptyState.tsx`)
- `features/*/components/` - Feature-specific components (e.g., `TaskCard.tsx`, `KanbanColumn.tsx`, `ChatMessage.tsx`)
- Naming: PascalCase for component files, match filename to component name
- One component per file (except tightly coupled sub-components)
- Keep shadcn/ui filenames as-is (lowercase like `button.tsx`)

```
components/
├── ui/                    # shadcn/ui (don't edit, lowercase)
│   ├── button.tsx
│   ├── card.tsx
│   └── dialog.tsx
├── ConfirmDialog.tsx      # Shared composite (PascalCase)
├── SearchInput.tsx
└── EmptyState.tsx

features/task/components/
├── TaskCard.tsx
├── TaskForm.tsx
└── KanbanBoard.tsx
```

---

## Question #12: Loading and Error State Patterns

How should loading and error states be implemented across the app?

### Answer

**Component-level with shared primitives**:

- Shared components: `LoadingSpinner.tsx`, `ErrorMessage.tsx`, `EmptyState.tsx`
- React Query's `isLoading`, `isError`, `error` drive component state
- Page-level skeleton loaders for initial loads (better perceived performance)
- Inline spinners for refetches and mutations (non-blocking)
- Toast notifications for mutation errors
- Error boundaries for unexpected crashes

```typescript
// Pattern in pages:
const { data, isLoading, isError, error } = useWorkspaces();

if (isLoading) return <WorkspaceListSkeleton />;
if (isError) return <ErrorMessage error={error} retry={refetch} />;
if (data.length === 0) return <EmptyState action={<CreateWorkspaceButton />} />;

return <WorkspaceGrid workspaces={data} />;
```

---

## Question #13: Optimistic Updates Strategy

Which actions should use optimistic updates vs. wait for server confirmation?

### Answer

**Selective optimistic updates for low-risk actions**:

**Optimistic (update UI immediately):**
- Adding a comment to a task (append to list, rollback if fails)
- Sending a chat message (append to conversation)
- Toggling task priority (visual badge change)
- Theme switching (instant visual feedback)

**Wait for server (show loading state):**
- Creating/deleting workspaces, tasks, agents, chats (data creation/destruction)
- Status changes (triggers backend logic, SSE events)
- Agent reordering (affects processing order)
- Form submissions with validation

React Query's `onMutate`/`onError`/`onSettled` pattern for optimistic updates.

---

## Question #14: Mobile-First Implementation Patterns

How should mobile-first be implemented in practice with Tailwind CSS?

### Answer

**Tailwind mobile-first with breakpoint conventions**:

- Default styles are mobile; use `sm:`, `md:`, `lg:` for larger screens
- Breakpoint usage convention:
  - Base (no prefix): Mobile phones (<640px)
  - `sm:`: Large phones / small tablets (≥640px)
  - `md:`: Tablets (≥768px)
  - `lg:`: Desktops (≥1024px)
- Component-specific patterns:
  - Kanban: Single column scroll on mobile, 4-column grid on `md:`+
  - Modals: Full-screen on mobile (`fixed inset-0`), centered dialog on `md:`+
  - Navigation: Bottom tabs or hamburger on mobile, sidebar on `lg:`+
  - Agent reorder: Up/down arrows on mobile, drag-and-drop on `md:`+ (per SESSION-010 Q#14)

```typescript
// Example: Kanban board
<div className="flex flex-col gap-4 md:grid md:grid-cols-4 md:gap-6">
```

Always reference TailwindCSS and shadcn/ui documentation for breakpoint values.

---

## Question #15: Frontend Testing Strategy

What testing approach should the frontend use?

### Answer

**Defer testing for initial development**: Focus on getting the app working first. Ensure linting and Prettier run before commits via pre-commit hooks. Testing strategy can be added later.

---

## Question #16: Linting and Formatting Configuration

What configuration and pre-commit setup should be used?

### Answer

**Standard tooling with pre-commit hooks**:

- **ESLint**: `@eslint/js` + `typescript-eslint` + `eslint-plugin-react-hooks` + `eslint-plugin-react-refresh`
- **Prettier**: Default config with minimal overrides
- **Import sorting**: `eslint-plugin-simple-import-sort` (same as backend per SESSION-012 Q#18)
- **Pre-commit**: `husky` + `lint-staged` to run ESLint and Prettier on staged files

```json
// package.json
{
  "lint-staged": {
    "*.{ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md,css}": ["prettier --write"]
  }
}
```

```javascript
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "trailingComma": "es5"
}
```

---

## Question #17: Type Sharing Between Frontend and Backend

Should types be shared between repositories, and how?

### Answer

**Manual duplication with discipline (for now)**:

- Frontend defines its own types in `features/*/types/` based on API responses
- Types are simpler on frontend (no database concerns, just what API returns)
- API response types documented in TECHNICAL_DESIGN.md serve as contract
- Revisit sharing when repo structure is finalized (SESSION-012 Q#14 noted monorepo vs separate repos is TBD)

```typescript
// features/workspace/types/workspace.types.ts
export interface Workspace {
  id: string;
  title: string;
  description: string;
  working_directory_mode: 'static' | 'temp';
  // ... only what frontend needs
}
```

---

## Question #18: Tailwind Styling Conventions

What conventions should be followed for consistent styling across the app?

### Answer

**shadcn/ui conventions + minimal customization**:

- Use shadcn/ui's CSS variables for theming (defined in `globals.css`)
- Use shadcn/ui's `cn()` utility for conditional classes (from `lib/utils.ts`)
- Prefer shadcn/ui components over custom implementations
- Custom colors/spacing: Extend Tailwind config only when necessary, use semantic names
- Avoid arbitrary values (`w-[347px]`) - use Tailwind's scale or define in config
- Class order: Follow Tailwind's recommended order (layout → spacing → typography → visual)
- Long class lists: Use `cn()` with logical grouping

```typescript
// Good: using cn() for readability and conditionals
<div className={cn(
  "flex items-center gap-2 p-4",
  "rounded-lg border bg-card",
  isActive && "border-primary"
)}>
```

---

## Question #19: Development Workflow and Proxy Setup

How should the development workflow be configured?

### Answer

**Vite proxy with concurrent dev servers**:

- Vite dev server runs on port 5173 (default)
- Vite config proxies `/api/*` to backend
- Backend runs separately on port 3456

```typescript
// vite.config.ts
export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': {
        target: process.env.VITE_API_BASE_URL || 'http://localhost:3456',
        changeOrigin: true,
      },
    },
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

```json
// package.json scripts
{
  "dev": "vite",
  "build": "tsc && vite build",
  "preview": "vite preview",
  "lint": "eslint . --ext ts,tsx --fix",
  "format": "prettier --write \"src/**/*.{ts,tsx}\""
}
```

---

## Question #20: File Naming Conventions

What naming conventions should apply across all file types?

### Answer

**Type-based naming convention**:

| File Type | Convention | Example |
|-----------|------------|---------|
| React components | PascalCase | `WorkspaceCard.tsx` |
| Hooks | kebab-case with `use-` prefix | `use-workspaces.ts` |
| API files | kebab-case with `.api` suffix | `workspace.api.ts` |
| Type files | kebab-case with `.types` suffix | `workspace.types.ts` |
| Utility files | kebab-case | `date-utils.ts` |
| Test files | Match source + `.test` | `WorkspaceCard.test.tsx`, `use-workspaces.test.ts` |
| shadcn/ui components | Keep as-is (lowercase) | `button.tsx`, `card.tsx` |
| Route config | kebab-case | `routes/index.tsx` |
| Stores (if added) | kebab-case with `-store` suffix | `theme-store.ts` |

---

## Question #21: Complete Frontend Structure Summary

The complete frontend repository structure consolidating all decisions.

### Answer

```
malamar-frontend/
├── src/
│   ├── App.tsx                      # Root component, providers
│   ├── main.tsx                     # Entry point
│   │
│   ├── routes/
│   │   └── index.tsx                # React Router configuration
│   │
│   ├── layouts/
│   │   ├── RootLayout.tsx
│   │   └── WorkspaceLayout.tsx
│   │
│   ├── components/                  # Shared components
│   │   ├── ui/                      # shadcn/ui (lowercase, don't edit)
│   │   │   ├── button.tsx
│   │   │   ├── card.tsx
│   │   │   └── ...
│   │   ├── ConfirmDialog.tsx        # Shared composites (PascalCase)
│   │   ├── EmptyState.tsx
│   │   ├── ErrorMessage.tsx
│   │   ├── LoadingSpinner.tsx
│   │   ├── Markdown.tsx
│   │   ├── SearchInput.tsx
│   │   ├── ThemeToggle.tsx
│   │   └── TimeAgo.tsx
│   │
│   ├── features/
│   │   ├── workspace/
│   │   │   ├── index.ts             # Public exports
│   │   │   ├── pages/
│   │   │   │   ├── WorkspaceList.tsx
│   │   │   │   └── WorkspaceDetail.tsx
│   │   │   ├── components/
│   │   │   │   ├── WorkspaceCard.tsx
│   │   │   │   ├── WorkspaceForm.tsx
│   │   │   │   └── WorkspaceSettings.tsx
│   │   │   ├── hooks/
│   │   │   │   ├── use-workspaces.ts
│   │   │   │   └── use-workspace.ts
│   │   │   ├── api/
│   │   │   │   └── workspace.api.ts
│   │   │   └── types/
│   │   │       └── workspace.types.ts
│   │   │
│   │   ├── task/
│   │   │   ├── pages/
│   │   │   │   └── TaskDetail.tsx
│   │   │   ├── components/
│   │   │   │   ├── KanbanBoard.tsx
│   │   │   │   ├── KanbanColumn.tsx
│   │   │   │   ├── TaskCard.tsx
│   │   │   │   ├── TaskForm.tsx
│   │   │   │   └── CommentList.tsx
│   │   │   ├── hooks/
│   │   │   ├── api/
│   │   │   └── types/
│   │   │
│   │   ├── chat/
│   │   │   ├── pages/
│   │   │   │   └── ChatDetail.tsx
│   │   │   ├── components/
│   │   │   │   ├── ChatList.tsx
│   │   │   │   ├── ChatMessage.tsx
│   │   │   │   └── MessageInput.tsx
│   │   │   ├── hooks/
│   │   │   ├── api/
│   │   │   └── types/
│   │   │
│   │   ├── agent/
│   │   │   ├── components/
│   │   │   │   ├── AgentList.tsx
│   │   │   │   └── AgentForm.tsx
│   │   │   ├── hooks/
│   │   │   ├── api/
│   │   │   └── types/
│   │   │
│   │   └── settings/
│   │       ├── pages/
│   │       │   └── GlobalSettings.tsx
│   │       ├── components/
│   │       ├── hooks/
│   │       ├── api/
│   │       └── types/
│   │
│   ├── hooks/                       # Shared hooks
│   │   ├── use-sse.ts
│   │   └── use-toast.ts
│   │
│   ├── lib/                         # Utilities
│   │   ├── api-client.ts
│   │   ├── query-client.ts
│   │   ├── sse-handlers.ts
│   │   ├── date-utils.ts
│   │   └── utils.ts                 # cn() helper from shadcn
│   │
│   ├── stores/                      # Zustand stores (when needed)
│   │
│   └── types/                       # Shared types
│       └── api.types.ts             # Common API types (error format, etc.)
│
├── public/                          # Static assets
│
├── index.html
├── vite.config.ts
├── tsconfig.json
├── tailwind.config.js
├── postcss.config.js
├── components.json                  # shadcn/ui config
├── .eslintrc.cjs
├── .prettierrc
├── package.json
└── bun.lockb
```

**Conventions:**
- Component files: PascalCase (`WorkspaceCard.tsx`)
- Hook files: kebab-case with `use-` prefix (`use-workspaces.ts`)
- Other files: kebab-case (`workspace.api.ts`, `workspace.types.ts`)
- shadcn/ui: Keep lowercase as generated
- Imports: Relative paths with `@/` alias for `src/`

---

## Question #22: Layout and Navigation Structure

How should layouts and navigation be structured?

### Answer

**Nested layouts with React Router**:

- `RootLayout.tsx` - App shell with header (logo, theme toggle, settings link), SSE connection
- `WorkspaceLayout.tsx` - Workspace-specific header (back button, workspace title), tabs (Tasks, Chat, Agents, Settings)
- Mobile: Bottom navigation or hamburger menu in header
- Desktop: Header with navigation links

```typescript
// routes/index.tsx
{
  path: '/',
  element: <RootLayout />,
  children: [
    { index: true, element: <WorkspaceList /> },
    { path: 'settings', element: <GlobalSettings /> },
    {
      path: 'workspaces/:workspaceId',
      element: <WorkspaceLayout />,
      children: [
        { index: true, element: <Navigate to="tasks" replace /> },
        { path: 'tasks', element: <KanbanBoard /> },
        { path: 'tasks/:taskId', element: <TaskDetail /> },  // Modal over tasks
        { path: 'chats', element: <ChatList /> },
        { path: 'chats/:chatId', element: <ChatDetail /> },
        { path: 'agents', element: <AgentList /> },
        { path: 'settings', element: <WorkspaceSettings /> },
      ],
    },
  ],
}
```

---

## Question #23: Accessibility Implementation

What accessibility considerations should be followed?

### Answer

**Leverage Radix + follow baseline practices**:

- **Trust Radix**: shadcn/ui components (Dialog, Dropdown, Tabs, etc.) handle ARIA, focus management, keyboard navigation
- **Semantic HTML**: Use proper elements (`button`, `nav`, `main`, `h1-h6`) instead of divs with roles
- **Focus management**: Ensure modals trap focus, return focus on close (Radix handles this)
- **Keyboard navigation**: All interactive elements reachable via Tab, activatable via Enter/Space
- **Color contrast**: Stick to shadcn/ui's theme which meets WCAG AA contrast
- **Loading states**: Use `aria-busy` for loading regions, announce to screen readers
- **Form errors**: Associate error messages with inputs via `aria-describedby`
- **Skip link**: Add "Skip to main content" for keyboard users (in RootLayout)

No need for extensive ARIA since Radix handles it. Focus on not breaking what Radix provides.

---

## Question #24: API Error Handling Patterns

How should the frontend handle API errors consistently?

### Answer

**Centralized error handling with user-friendly messages**:

- API client parses error responses and throws typed errors
- React Query's `onError` callbacks handle mutation errors (show toast)
- Query errors handled at component level (show ErrorMessage component)
- Error messages displayed directly from `error.message` (backend provides user-friendly text per SESSION-011 Q#7)
- Network errors (no response) get generic "Connection error" message

```typescript
// lib/api-client.ts
class ApiError extends Error {
  constructor(public code: string, message: string, public status: number) {
    super(message);
  }
}

async function handleResponse<T>(response: Response): Promise<T> {
  if (!response.ok) {
    const body = await response.json().catch(() => ({}));
    throw new ApiError(
      body.error?.code ?? 'UNKNOWN',
      body.error?.message ?? 'An unexpected error occurred',
      response.status
    );
  }
  return response.json();
}

// In mutation:
const mutation = useMutation({
  mutationFn: workspaceApi.create,
  onError: (error) => {
    toast.error(error.message);
  },
});
```

---

## Question #25: Date/Time Formatting Implementation

How should date/time formatting be implemented?

### Answer

**Lightweight utility with Intl API**:

- Custom `formatRelativeTime()` function in `lib/date-utils.ts`
- Uses native `Intl.RelativeTimeFormat` and `Intl.DateTimeFormat` (no library needed)
- Returns relative strings per SESSION-010 Q#17 rules
- Component for display with tooltip showing absolute time

```typescript
// lib/date-utils.ts
export function formatRelativeTime(date: Date | string): string {
  const d = new Date(date);
  const now = new Date();
  const diffMs = now.getTime() - d.getTime();
  const diffMins = Math.floor(diffMs / 60000);
  const diffHours = Math.floor(diffMs / 3600000);
  const diffDays = Math.floor(diffMs / 86400000);
  
  if (diffMins < 1) return 'just now';
  if (diffMins < 60) return `${diffMins} min ago`;
  if (diffHours < 24) return `${diffHours} hours ago`;
  if (diffDays < 7) return `${diffDays} days ago`;
  
  // Absolute format for older dates
  const formatter = new Intl.DateTimeFormat(undefined, {
    month: 'short',
    day: 'numeric',
    year: d.getFullYear() !== now.getFullYear() ? 'numeric' : undefined,
  });
  return formatter.format(d);
}

// components/TimeAgo.tsx
export function TimeAgo({ date }: { date: string }) {
  const absolute = new Intl.DateTimeFormat(undefined, {
    dateStyle: 'full',
    timeStyle: 'short',
  }).format(new Date(date));
  
  return (
    <span title={absolute}>{formatRelativeTime(date)}</span>
  );
}
```

---

## Question #26: Toast Notification Implementation

How should toasts be implemented?

### Answer

**shadcn/ui Sonner integration**:

- shadcn/ui provides a toast component built on Sonner (modern toast library)
- Configure with SESSION-010 Q#11 requirements
- Export `toast` function from shared hook for easy access
- SSE handlers and mutation callbacks use the same toast API

```typescript
// Install shadcn toast (uses Sonner under the hood)
// npx shadcn-ui@latest add sonner

// App.tsx - add Toaster with config
import { Toaster } from '@/components/ui/sonner';

<Toaster
  position="bottom-right"
  duration={5000}
  visibleToasts={3}
  closeButton
/>

// Usage anywhere:
import { toast } from 'sonner';

// Success
toast.success('Workspace created');

// Error
toast.error('Failed to create workspace');

// Clickable with action
toast('Task moved to In Review', {
  action: {
    label: 'View',
    onClick: () => navigate(`/workspaces/${wsId}/tasks/${taskId}`),
  },
});
```

---

## Question #27: Theme Implementation

How should theming be implemented?

### Answer

**shadcn/ui theme pattern with next-themes**:

- Use `next-themes` library (works with Vite/React, not just Next.js)
- Provides `ThemeProvider`, `useTheme` hook, handles localStorage and system preference
- shadcn/ui's CSS variables automatically respond to `.dark` class on `<html>`
- Theme toggle component in header

```typescript
// App.tsx
import { ThemeProvider } from 'next-themes';

<ThemeProvider attribute="class" defaultTheme="system" enableSystem>
  <RouterProvider router={router} />
  <Toaster />
</ThemeProvider>

// components/ThemeToggle.tsx
import { useTheme } from 'next-themes';
import { Moon, Sun } from 'lucide-react';

export function ThemeToggle() {
  const { theme, setTheme } = useTheme();
  
  return (
    <Button variant="ghost" size="icon" onClick={() => 
      setTheme(theme === 'dark' ? 'light' : 'dark')
    }>
      <Sun className="h-4 w-4 rotate-0 scale-100 transition-all dark:-rotate-90 dark:scale-0" />
      <Moon className="absolute h-4 w-4 rotate-90 scale-0 transition-all dark:rotate-0 dark:scale-100" />
    </Button>
  );
}
```

---

## Question #28: Drag-and-Drop Implementation

What library should be used for drag-and-drop (agent reordering)?

### Answer

**@dnd-kit**:

- Modern, lightweight, accessible drag-and-drop
- Built for React with hooks-based API
- Supports sortable lists (agent reordering) and grid layouts (future Kanban)
- Keyboard accessible (important for a11y)
- Tree-shakeable, only import what's needed

```typescript
// Agent reordering example
import { DndContext, closestCenter } from '@dnd-kit/core';
import { SortableContext, verticalListSortingStrategy } from '@dnd-kit/sortable';

<DndContext collisionDetection={closestCenter} onDragEnd={handleDragEnd}>
  <SortableContext items={agents} strategy={verticalListSortingStrategy}>
    {agents.map((agent) => (
      <SortableAgentItem key={agent.id} agent={agent} />
    ))}
  </SortableContext>
</DndContext>
```

Note: SESSION-010 Q#14 specified "No drag-and-drop between columns" for Kanban tasks - status changes happen in task detail. So dnd-kit is primarily for agent reordering.

---

## Question #29: Markdown Rendering

How should markdown content be rendered in the UI?

### Answer

**react-markdown with remark/rehype plugins**:

- `react-markdown` for rendering markdown to React components
- `remark-gfm` for GitHub Flavored Markdown (tables, strikethrough, task lists)
- `rehype-sanitize` for XSS protection (sanitize HTML in markdown)
- Custom components map to use shadcn/ui typography styles

```typescript
// components/Markdown.tsx
import ReactMarkdown from 'react-markdown';
import remarkGfm from 'remark-gfm';
import rehypeSanitize from 'rehype-sanitize';

const components = {
  h1: ({ children }) => <h1 className="text-2xl font-bold mt-6 mb-4">{children}</h1>,
  h2: ({ children }) => <h2 className="text-xl font-semibold mt-4 mb-2">{children}</h2>,
  p: ({ children }) => <p className="mb-4">{children}</p>,
  code: ({ inline, children }) => 
    inline 
      ? <code className="bg-muted px-1 py-0.5 rounded">{children}</code>
      : <pre className="bg-muted p-4 rounded-lg overflow-x-auto"><code>{children}</code></pre>,
  // ... more mappings
};

export function Markdown({ content }: { content: string }) {
  return (
    <ReactMarkdown 
      remarkPlugins={[remarkGfm]} 
      rehypePlugins={[rehypeSanitize]}
      components={components}
    >
      {content}
    </ReactMarkdown>
  );
}
```

---

## Question #30: Code Syntax Highlighting

Should code within markdown have syntax highlighting?

### Answer

**rehype-highlight with highlight.js**:

- Add `rehype-highlight` plugin to react-markdown
- Uses highlight.js for syntax highlighting (supports 190+ languages)
- Import a theme CSS that works with both light and dark modes
- Relatively lightweight (~30KB for common languages)

```typescript
// components/Markdown.tsx
import rehypeHighlight from 'rehype-highlight';
import 'highlight.js/styles/github-dark.css'; // or a theme that adapts

<ReactMarkdown 
  remarkPlugins={[remarkGfm]} 
  rehypePlugins={[rehypeSanitize, rehypeHighlight]}
  components={components}
>
  {content}
</ReactMarkdown>
```

```css
/* globals.css - theme-aware code blocks */
.hljs {
  background: hsl(var(--muted));
}
```

---

## Question #31: Icon Library

What icon library should be used?

### Answer

**Lucide React**:

- shadcn/ui uses Lucide icons by default (already in the ecosystem)
- Tree-shakeable - only imports icons you use
- Consistent stroke-based style
- Large icon set (1000+ icons)
- React components with props for size, color, strokeWidth

```typescript
import { Plus, Trash, Settings, MessageSquare, Check } from 'lucide-react';

<Button>
  <Plus className="h-4 w-4 mr-2" />
  Create Workspace
</Button>
```

---

## Question #32: Environment Variables Handling

How should environment variables be handled?

### Answer

**Vite's built-in env handling with minimal usage**:

- Always use relative URLs in API client code (`/api/...`)
- Only use `VITE_API_BASE_URL` in `vite.config.ts` to configure the proxy target for non-default port scenarios
- No environment variable references in application code

```typescript
// lib/api-client.ts - always relative, no env var
const apiClient = {
  get: <T>(path: string) => fetch(`/api${path}`).then(handleResponse<T>),
  // ...
};
```

```typescript
// vite.config.ts - env var only here for proxy target
export default defineConfig({
  server: {
    proxy: {
      '/api': {
        target: process.env.VITE_API_BASE_URL || 'http://localhost:3456',
        changeOrigin: true,
      },
    },
  },
});
```

Malamar's architecture (frontend served by backend) means relative URLs work everywhere - dev (Vite proxy) and production (same origin).
