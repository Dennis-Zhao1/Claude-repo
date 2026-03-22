# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator. Users describe components in natural language; Claude generates them in real-time with a live preview and editable code editor. Built with Next.js 15 (App Router), React 19, Tailwind CSS v4, Prisma/SQLite, and the Anthropic Claude API via Vercel AI SDK.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Dev server with Turbopack
npm run dev:daemon   # Dev server in background (logs to logs.txt)
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest
npm run db:reset     # Reset database (destructive)
```

Run a single test file: `npx vitest run src/lib/__tests__/file-system.test.ts`

Environment: copy `.env` and set `ANTHROPIC_API_KEY`. Without it, the app falls back to a mock provider that returns static components.

## Architecture

### Core Data Flow

1. User sends a message in the chat UI
2. `ChatInterface` sends the conversation + serialized VirtualFileSystem to `POST /api/chat`
3. The API endpoint calls Claude with two tools: `str_replace_editor` (create/edit files) and `file_manager` (rename/delete)
4. Claude's tool calls stream back; the client applies them to the in-memory VFS
5. `PreviewFrame` picks up VFS changes, transpiles JSX via Babel, and renders in a sandboxed iframe
6. On completion, if the user is authenticated, the project (messages + VFS snapshot) is saved to SQLite

### Virtual File System (`src/lib/file-system.ts`)

All generated files live in memory — no disk I/O for component files. The `VirtualFileSystem` class manages a tree of nodes, supports CRUD + rename, and serializes to/from JSON for database persistence. It is exposed app-wide via `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`).

### AI Integration (`src/app/api/chat/route.ts`)

- Uses `streamText()` from Vercel AI SDK with `anthropic` provider
- System prompt defined in `src/lib/prompts/generation.tsx`; it instructs Claude to use `/App.jsx` as the entry point and Tailwind CSS for styling
- Prompt caching via Anthropic's ephemeral `cacheControl` on the system prompt
- Limits: `maxTokens: 10000`, `maxSteps: 40` (4 for mock provider)
- Tool schemas defined in `src/lib/tools/`

### Live Preview (`src/components/preview/PreviewFrame.tsx`)

- Renders inside a sandboxed `<iframe>`
- Entry point is looked up as `App.jsx` / `App.tsx` / `index.jsx` / `index.tsx`
- JSX is transpiled in-browser by Babel (`src/lib/transform/jsx-transformer.ts`), which strips CSS imports and resolves `@/` alias imports to blob URLs
- React and React-DOM are loaded from `esm.sh` via import maps

### Authentication (`src/lib/auth.ts`, `src/actions/`)

- JWT sessions stored in HTTP-only cookies (7-day expiry), implemented with `jose`
- Passwords hashed with `bcrypt`
- Auth is optional — anonymous sessions are tracked in localStorage via `anon-work-tracker.ts`
- Server actions in `src/actions/` handle sign-up, sign-in, sign-out, and project CRUD

### Database (`prisma/schema.prisma`)

The schema is the authoritative reference for database structure. Two models: `User` and `Project`. Projects store `messages` (JSON array) and `data` (serialized VFS) as JSON columns in SQLite. Cascade-deletes on user removal.

## Key Conventions

- Path alias `@/*` maps to `src/*`
- shadcn/ui components live in `src/components/ui/`; use the `new-york` style with CSS variables
- State management is React Context only (no Redux/Zustand)
- Tests use Vitest + jsdom + React Testing Library; test files sit next to source in `__tests__/` subdirectories
- Use comments sparingly — only comment complex code
- nothing