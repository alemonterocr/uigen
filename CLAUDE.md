# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run test         # Run Vitest tests
npm run db:reset     # Reset the SQLite database
```

To run a single test file:

```bash
npx vitest run src/path/to/test.test.ts
```

Environment variable required: `ANTHROPIC_API_KEY` in `.env`. Without it, the app runs in demo mode using a `MockLanguageModel`.

## Architecture

UIGen is an AI-powered React component generator. Users chat with Claude, which generates/edits React files in a virtual file system. Changes appear live in an iframe preview.

### Core Data Flow

1. User sends message → `ChatInterface` → `/api/chat` (POST)
2. API calls Claude via Vercel AI SDK `streamText()` with two tools: `str_replace_editor` and `file_manager`
3. Tool calls are streamed back; `ChatProvider` routes them to `FileSystemProvider.handleToolCall()`
4. `FileSystemProvider` updates the in-memory `VirtualFileSystem` instance
5. `PreviewFrame` picks up file changes, transpiles JSX via Babel standalone with an esm.sh import map, and injects new `srcdoc` into an iframe

### Virtual File System

`src/lib/file-system.ts` — an in-memory tree (not disk). All AI-generated files live here. The filesystem serializes to JSON for storage in the `Project.data` column (SQLite via Prisma).

### State Management

Two React contexts wrap the entire app:

- **`FileSystemProvider`** (`src/lib/contexts/file-system-context.tsx`): owns the `VirtualFileSystem` instance, selected file, and handles AI tool call execution
- **`ChatProvider`** (`src/lib/contexts/chat-context.tsx`): wraps Vercel AI SDK's `useChat()`, manages message streaming, and forwards tool calls to `FileSystemProvider`

### AI Integration

- **Provider**: `src/lib/provider.ts` — returns `claude-haiku-4-5` if `ANTHROPIC_API_KEY` is set, otherwise a `MockLanguageModel`
- **API route**: `src/app/api/chat/route.ts` — uses `streamText()` with prompt caching headers; saves conversation + file state to DB on completion
- **Tools**: `src/lib/tools/str-replace.ts` (view/create/str_replace/insert commands) and `src/lib/tools/file-manager.ts` (rename/delete)
- **System prompt**: `src/lib/prompts/generation.tsx` — instructs Claude to use `/App.jsx` as the entry point and `@/` for local imports

### Preview Pipeline

`src/lib/transform/jsx-transformer.ts` uses Babel standalone to transpile JSX, detects imports, builds an esm.sh import map, and returns full HTML. `PreviewFrame` auto-detects the entry file (`App.jsx`, `index.jsx`, etc.) and renders it in an iframe via `srcdoc`.

### Authentication

JWT sessions via `jose` (httpOnly cookies, 7-day expiry). `src/middleware.ts` protects API routes. Projects can be anonymous (no `userId`) or owned. Server actions in `src/actions/` handle sign-up, sign-in, sign-out, and project CRUD.

### Layout

Three-panel split via `react-resizable-panels`: Chat (left 35%) | Preview or Code editor (right 65%). Code view further splits into FileTree (30%) + Monaco editor (70%).

### Development Best Practices

- Use comments sparingly. Onlycomment complex code.

### Database

- The database schema is defined in the @prisma/schema.prisma file. Reference it anytime you need to understand the structure of data stored in the database.
