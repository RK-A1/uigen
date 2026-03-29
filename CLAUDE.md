# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development
npm run dev          # Start dev server (Turbopack) with Node compat layer
npm run build        # Production build
npm run start        # Start production server

# Database
npm run setup        # Install deps + generate Prisma client + run migrations
npm run db:reset     # Reset database (destructive)
npx prisma studio    # Browse database

# Testing
npm test             # Run all tests with Vitest
npm test -- src/components/chat/__tests__/ChatInterface.test.tsx  # Run single test file

# Lint
npm run lint         # ESLint via Next.js
```

**Environment variables** (`.env`):
- `ANTHROPIC_API_KEY` — optional; omitting it activates the mock provider with static component output
- `JWT_SECRET` — defaults to `development-secret-key` if unset

## Architecture

UIGen is an AI-powered React component generator. Users describe components in chat; Claude generates/edits files in a virtual file system; a sandboxed iframe previews the result in real time.

### Request Flow

1. User message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. `streamText()` (Vercel AI SDK) calls Claude with two tools: `str_replace_editor` and `file_manager`
3. Tool calls are streamed back → `FileSystemContext` applies them to the in-memory `VirtualFileSystem`
4. `PreviewFrame` Babel-transforms the virtual FS files → generates an import map + blob URL → injects into a sandboxed `<iframe>`
5. On stream completion, chat history + serialized FS are saved to Prisma (`Project.messages`, `Project.data`)

### State Management

Two React contexts wrap the entire app (`src/lib/contexts/`):

- **`FileSystemContext`** — owns the `VirtualFileSystem` instance; handles tool-call results from AI; tracks `selectedFile`
- **`ChatContext`** — wraps Vercel AI SDK's `useChat`; connects to `/api/chat`; bridges tool results back to the file system; tracks anonymous work for unsaved-session recovery

### Virtual File System (`src/lib/file-system.ts`)

All "files" live in memory only — nothing is written to disk. The FS is serialized to JSON for storage in the `Project.data` column. Entry point detection looks for `App.jsx`, `App.tsx`, `index.jsx`, etc.

### AI Provider (`src/lib/provider.ts`)

- With `ANTHROPIC_API_KEY`: uses Claude via `@ai-sdk/anthropic` with prompt caching (Anthropic ephemeral `cacheControl`)
- Without it: falls back to `MockLanguageModel` which returns static Counter/Form/Card components — useful for UI development without API costs

### Component Preview (`src/lib/transform/jsx-transformer.ts`)

Babel standalone transforms JSX → ES modules. An import map is generated mapping bare specifiers (`react`, `react-dom`, internal `@/` paths) to blob URLs or `esm.sh` CDN URLs. The result is injected into a sandboxed `<iframe>` with `allow-scripts` only.

### Authentication

JWT sessions via `jose`, stored as HttpOnly cookies (7-day expiry). Passwords hashed with bcrypt. `src/middleware.ts` guards `/api/projects` and `/api/filesystem` routes. Server actions in `src/actions/` handle sign-up, sign-in, sign-out, and project CRUD.

### Database

Prisma + SQLite (`prisma/dev.db`). Two models:
- `User` — email/password
- `Project` — `messages` (JSON string), `data` (JSON string of virtual FS), optional `userId` (anonymous projects have `null`)

Generated client outputs to `src/generated/prisma/` — don't edit those files.

### Key Directories

| Path | Purpose |
|------|---------|
| `src/app/api/chat/route.ts` | Streaming AI endpoint with tool definitions |
| `src/lib/contexts/` | Global state (FS + chat) |
| `src/lib/tools/` | Tool implementations: `str-replace.ts`, `file-manager.ts` |
| `src/lib/prompts/generation.tsx` | System prompt sent to Claude |
| `src/lib/transform/` | JSX → iframe pipeline |
| `src/components/preview/` | `PreviewFrame` (iframe renderer) |
| `src/components/editor/` | Monaco editor + file tree |
| `src/components/chat/` | Chat UI components |
| `src/actions/` | Next.js Server Actions for auth + projects |

### Testing

Tests use Vitest + jsdom + Testing Library. Test files live in `__tests__/` subdirectories alongside the code they test. The mock provider is the preferred approach for unit tests that touch the AI pipeline.
