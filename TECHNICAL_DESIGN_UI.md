# Technical Design: UI Implementation

This document covers the frontend implementation details for Malamar's web interface using React.

For UX design decisions and mockups, see [TECHNICAL_DESIGN_UX.md](./TECHNICAL_DESIGN_UX.md).
For product specifications (what and why), see [SPECS.md](./SPECS.md).

---

## Table of Contents

- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [State Management](#state-management)
- [Data Fetching](#data-fetching)
- [Real-Time Updates](#real-time-updates)
- [Component Architecture](#component-architecture)
- [Build and Distribution](#build-and-distribution)

---

## Tech Stack

- **Framework**: React (via create-vite-app)
- **Runtime/Build**: Bun
- **Language**: TypeScript

*TBD: Additional libraries (routing, UI components, etc.)*

---

## Project Structure

*TBD: Directory structure and file organization*

---

## State Management

*TBD: State management approach (React Query, Zustand, Context, etc.)*

---

## Data Fetching

### Polling Strategy

- Task detail view polls every 3 seconds
- Uses React Query (or similar) for caching and deduplication

*TBD: Detailed data fetching patterns*

---

## Real-Time Updates

### SSE Connection

- Single SSE connection to `/api/events`
- Client-side filtering by workspace if needed
- Toast notifications for relevant events

*TBD: SSE connection management and event handling*

---

## Component Architecture

*TBD: Key components and their responsibilities*

### Pages

- Workspace List
- Workspace Detail
- Workspace Settings
- Global Settings

### Shared Components

*TBD: Reusable components (modals, forms, etc.)*

---

## Build and Distribution

### Single Executable Distribution

**Preferred approach:** Bundle frontend build output directly into Bun binary using `bun build --compile`, serving embedded static assets if supported.

**Fallback approach:**
1. Compress UI dist into binary
2. On startup, decompress to `~/.malamar/ui/`
3. Serve static files from that location
4. Overwrite on each startup to ensure UI matches binary version

---

## Decisions Log

*Use this section to track implementation decisions as they are made.*

| Date | Decision | Rationale |
|------|----------|-----------|
| TBD | TBD | TBD |