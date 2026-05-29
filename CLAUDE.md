# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**UIGen** — an AI-powered React component generator with live preview. Users describe UI in natural language; Claude generates components using tool calls that write to a virtual file system, displayed in a live preview panel.

## Commands

```bash
npm run setup        # One-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server (Turbopack) at http://localhost:3000
npm run build        # Production build
npm run lint         # ESLint via Next.js
npm run test         # Vitest (unit + component tests)
npm run db:reset     # Force-reset SQLite database
```

To run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

## Environment

Set `ANTHROPIC_API_KEY` to use real Claude AI. If unset, the app falls back to a mock provider automatically (see `src/lib/provider.ts`).

## Architecture

### AI Code Generation Flow

1. User sends a message via `ChatInterface` → POST to `/api/chat`
2. The route uses Vercel AI SDK `streamText()` with two tools:
   - `str_replace_editor` (`src/lib/tools/str-replace.ts`) — edit existing file content
   - `file_manager` (`src/lib/tools/file-manager.ts`) — create/delete files
3. Tool calls mutate the **virtual file system** (in-memory, never on disk)
4. `VirtualFileSystem` class (`src/lib/file-system.ts`) serializes state to JSON stored in Prisma's `project.data` field
5. `PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) compiles JSX in-browser via `@babel/standalone` and renders the live preview

### State Management

Two React contexts (no Redux/Zustand):
- `FileSystemProvider` (`src/lib/contexts/file-system-context.tsx`) — virtual file system state
- `ChatProvider` (`src/lib/contexts/chat-context.tsx`) — message history

### Layout

Three-panel resizable layout via `react-resizable-panels` in `src/app/main-content.tsx`:
- Left (35%): Chat
- Right top: Preview iframe / Code tabs
- Right code view: File tree (30%) + Monaco editor (70%)

### Auth & Sessions

- JWT sessions via `jose`, bcrypt password hashing
- Server actions in `src/actions/` for signup/login/logout
- `src/middleware.ts` guards routes and validates sessions
- `src/lib/auth.ts` handles `getSession` / `signToken`

### Data Model (Prisma + SQLite)

```
User  ─< Project
Project.messages  — JSON array of chat messages
Project.data      — serialized VirtualFileSystem JSON
```

### AI Configuration (src/app/api/chat/route.ts)

- Model: `claude-haiku-4-5` (via `@ai-sdk/anthropic`)
- Max tokens: 10,000 | Max steps: 40 (real) / 4 (mock)
- Max route duration: 120s
- System prompt with ephemeral cache control (prompt caching enabled)
- System prompt template: `src/lib/prompts/generation.ts`

## Key Conventions

- Path alias `@/*` maps to `./src/*`
- shadcn/ui components live in `src/components/ui/` (new-york style, Lucide icons)
- Server-only code uses the `server-only` package for import protection
- Tests use Vitest + jsdom + Testing Library; test files in `__tests__/` subdirectories alongside source
