# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface; Claude generates code using tool calls; code renders live in a sandboxed iframe.

## Commands

```bash
# Initial setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (with Turbopack)
npm run dev

# Build
npm run build

# Lint
npm run lint

# Run all tests
npm test

# Run a single test file
npx vitest src/lib/__tests__/file-system.test.ts

# Reset database
npm run db:reset

# Regenerate Prisma client after schema changes
npx prisma generate

# Create a new migration
npx prisma migrate dev
```

## Architecture

### Request Flow

1. User sends a message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. API reconstructs the virtual file system from serialized state sent by the client
3. Vercel AI SDK streams Claude's response; Claude uses two tools: `str_replace_editor` and `file_manager`
4. Tool calls stream back to the client and are processed by `FileSystemContext`
5. Updated files are Babel-transformed (JSX→JS) and loaded into the preview iframe via blob URLs + import maps
6. If authenticated, the project (messages + file system) is persisted to SQLite via Prisma

### Key Abstractions

**Virtual File System** (`src/lib/file-system.ts`): In-memory only — nothing is written to disk. Serializes/deserializes to JSON for database storage and for sending with each chat message.

**FileSystemContext** (`src/lib/contexts/file-system-context.tsx`): React context wrapping the virtual FS. Handles tool call execution from the AI (`str_replace`, `file_manager` operations) and propagates updates to the editor and preview.

**ChatContext** (`src/lib/contexts/chat-context.tsx`): Wraps Vercel AI SDK's `useChat`. Attaches serialized file system state to every outgoing message so the API can reconstruct it server-side.

**JSX Transformer** (`src/lib/transform/jsx-transformer.ts`): Uses `@babel/standalone` to transform JSX to browser-runnable JS. Generates blob URLs for each file and builds an import map so modules can resolve each other dynamically.

**PreviewFrame** (`src/components/preview/PreviewFrame.tsx`): Sandboxed iframe that auto-discovers the entry point (`App.jsx`, `index.tsx`, etc.) from the virtual file system and loads it via the import map.

**Language Model Provider** (`src/lib/provider.ts`): Returns the real Anthropic Claude model if `ANTHROPIC_API_KEY` is set in `.env`; otherwise returns a `MockLanguageModel` that produces static but realistic tool-call responses.

**Authentication** (`src/lib/auth.ts`): JWT in HttpOnly cookies (7-day expiry) using `jose`. Server-only module. Anonymous users can use the app without signing in; their work is tracked via `anon-work-tracker.ts`.

### Layout

Three-panel layout defined in `src/app/main-content.tsx`:
- Left (35%): Chat panel (`ChatInterface`)
- Right (65%): Toggles between live preview (`PreviewFrame`) and code view (`FileTree` + `CodeEditor`)

### Database

SQLite via Prisma. Two models: `User` and `Project`. `Project.messages` and `Project.fileSystem` are stored as JSON. Projects support optional `userId` (anonymous projects have no owner).

### Path Alias

`@/` maps to `src/` throughout the codebase.

## Environment Variables

- `ANTHROPIC_API_KEY` — optional; app runs with mock responses if absent
- `JWT_SECRET` — required for auth token signing
- `DATABASE_URL` — SQLite path (defaults to `./prisma/dev.db`)

## Testing

Tests use Vitest + jsdom + React Testing Library. Test files live in `__tests__/` subdirectories alongside the code they test. No special setup file — jsdom environment is configured in `vitest.config.mts`.
