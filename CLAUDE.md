# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # Install deps, generate Prisma client, run migrations (first-time setup)
npm run dev          # Dev server with Turbopack (node compat)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest unit tests
npm run db:reset     # Reset database to initial state
```

Single test file: `npx vitest run src/path/to/__tests__/file.test.ts`

## Environment

Copy `.env.example` to `.env` and add `ANTHROPIC_API_KEY`. Without it, the app falls back to a mock provider returning static components.

## Architecture

UIGen is an AI-powered React component generator: users describe components in natural language, Claude generates code via streaming tool calls, and results render instantly in a sandboxed iframe preview.

### Request Flow

1. User sends message → `ChatInterface` → Vercel AI SDK posts to `/api/chat`
2. `/api/chat` calls Claude (claude-haiku-4-5) with `str_replace_editor` and `file_manager` tools
3. Tool calls stream back → `ChatProvider` intercepts them → `FileSystemContext` processes file changes
4. `PreviewFrame` watches `FileSystemContext`, transforms JSX via Babel standalone, renders in iframe via srcdoc/blob URLs

### State Management

Two React Contexts own all runtime state:

- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`): wraps Vercel AI SDK's `useChat`, manages messages/input/status, auto-saves to DB on completion, tracks anonymous work in localStorage
- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`): manages the virtual in-memory file tree, processes incoming AI tool calls, triggers preview refresh

### Virtual File System

`src/lib/file-system.ts` is a pure in-memory Map-based tree. Files use absolute paths (e.g., `/App.jsx`, `/components/Card.jsx`). No disk I/O — serialized to JSON for database persistence.

### AI Tools

Defined in `src/lib/tools/`: `str_replace_editor` (create/update files with string replacement) and `file_manager` (rename/delete). These are the only mechanisms by which Claude modifies the virtual file system.

### Preview Rendering

`src/components/preview/` uses `@babel/standalone` to transform JSX/TypeScript to JS, builds import maps pointing to blob URLs for virtual modules, and injects everything into a sandboxed iframe (`allow-scripts allow-same-origin allow-forms`). Entry point detection order: `App.jsx → App.tsx → index.jsx → index.tsx`.

### Persistence

- **Authenticated users**: messages + file system serialized as JSON strings in SQLite via Prisma (`src/generated/prisma`). Models: `User`, `Project`.
- **Anonymous users**: stored in localStorage via `anon-work-tracker`; migrated to DB on sign-in.

### Routing

- `/` — home (redirects authenticated users to their projects)
- `/[projectId]` — project workspace (auth required)
- `/api/chat` — streaming AI endpoint

### Auth

JWT sessions in HTTP-only cookies (7-day expiry). Server Actions in `src/actions/` handle sign-up/sign-in (bcrypt) and project CRUD. Session verification is server-side via `src/lib/auth.ts`.

### Key Libraries

| Purpose | Library |
|---|---|
| AI streaming | Vercel AI SDK (`ai` + `@ai-sdk/anthropic`) |
| Code editor | Monaco Editor (`@monaco-editor/react`) |
| UI primitives | shadcn/ui (New York style, Tailwind v4, Radix UI) |
| JSX transform | `@babel/standalone` |
| ORM | Prisma (SQLite) |
| Testing | Vitest + jsdom + React Testing Library |
