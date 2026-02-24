# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server (Turbopack) on http://localhost:3000
npm run build        # Production build
npm start            # Start production server
npm run lint         # Run ESLint
npm run test         # Run Vitest tests
npm run setup        # One-time setup: install deps, generate Prisma, run migrations
npm run db:reset     # Reset SQLite database
```

To run a single test file: `npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx`

## Environment

Requires an `.env` file with `ANTHROPIC_API_KEY`. Without it, the app falls back to `MockLanguageModel` returning static responses — useful for UI development without API costs.

## Architecture

UIGen is an AI-powered React component generator. Users describe components in natural language; Claude generates them with real-time preview and code editing.

### Request Flow

1. User sends a message → `ChatProvider` (wraps Vercel AI SDK's `useChat`)
2. POST to `/api/chat` → `streamText()` with Claude (or mock model)
3. Claude responds with tool calls: `str_replace_editor` (create/edit files) or `file_manager` (rename/delete)
4. `FileSystemProvider` interprets tool calls and updates in-memory virtual FS
5. `PreviewFrame` re-renders: Babel transforms JSX in-browser, import maps resolve deps from `esm.sh` CDN
6. If authenticated and project exists, `onFinish` in route handler persists to SQLite via Prisma

### Virtual File System (`src/lib/file-system.ts`)

All file state lives in memory — nothing is written to disk. `FileSystemProvider` (`src/lib/contexts/file-system-context.tsx`) exposes `createFile`, `updateFile`, `deleteFile`, `renameFile`, `getFileContent`. The FS serializes to JSON for DB storage (`Project.data` column) and is sent with each API request so Claude has full context.

### AI Integration (`src/app/api/chat/route.ts`)

- Uses `streamText()` from Vercel AI SDK with `claude-haiku-4-5` model
- Two tools registered: `str_replace_editor` and `file_manager`
- System prompt lives in `src/lib/prompts/generation.tsx`
- Ephemeral prompt caching on system instructions
- `maxDuration = 120` seconds

### JSX Preview (`src/lib/transform/jsx-transformer.ts`)

Babel Standalone runs in-browser to transform JSX/TS. Generates an HTML srcdoc with an import map pointing to `esm.sh` for React 19 and other deps. Auto-detects entry point in order: `App.jsx → App.tsx → index.jsx → index.tsx`. CSS imports are stripped and replaced with blob URLs.

### Auth (`src/lib/auth.ts`, `src/actions/index.ts`)

JWT sessions in httpOnly cookies (7-day expiry), bcrypt password hashing. Anonymous use is allowed — `projectId` is optional. Authenticated users get project persistence via `[projectId]` route.

### Context Hierarchy

```
FileSystemProvider
  └── ChatProvider
        └── MainContent (3-panel resizable layout: Chat | Preview/Editor)
```

Consumers use `useFileSystem()` and `useChat()` hooks.

## Key Conventions

- Path alias: `@/*` → `src/*`
- Server actions in `src/actions/` use `"use server"`; interactive components use `"use client"`
- shadcn/ui primitives in `src/components/ui/`; feature components in domain folders (`auth/`, `chat/`, `editor/`, `preview/`)
- Tests co-located in `__tests__/` subdirectories, using jsdom + `@testing-library/react`
- Prisma client generated to `src/generated/prisma` (not the default location)
